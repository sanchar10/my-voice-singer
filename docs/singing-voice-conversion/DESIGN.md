# Local Singing Voice Conversion: Technical Design

Version: 0.3 draft, repository handoff

Date: 2026-09-25, Pacific time

Status: Proposed architecture. Interfaces below belong to a possible future application, not existing upstream APIs. Nothing here has been implemented or run on the user's GPU.

Companion documents: [Windows handoff](WINDOWS_HANDOFF.md) | [Product specification](SPEC.md) | [Research and sources](RESEARCH.md) | [Repository overview](../../README.md)

**Terminology:** source/original is the singing performance to convert; target voice is the user's own identity learned from their recordings. Source pitch, octave, timing, lyrics, and breath locations must not be replaced with the target recordings' performance.

## 1. Architecture decision

Use **Applio with a target-specific, F0-enabled RVC model**, not a new ML model. A thin wrapper is optional and should address only missing workflow features.

Initial engine: an Applio/RVC F0-enabled target-specific model. Begin with the upstream UI/CLI during the experiment; do not build the application until the quality gate passes.

The user can record 5-10 minutes of singing, so reference-only enrollment is no longer the reason to choose the primary engine. If the existing UI meets the workflow requirements, using it directly is a valid final solution. SoulX-Singer-SVC is a contingency only if the selected RVC workflow fails the quality goals after a bounded data/settings iteration or enrollment needs change; do not implement both initially.

The following stack and diagram are **deferred design**, not the Windows pilot's implementation checklist. Use the [handoff](WINDOWS_HANDOFF.md) for the current existing-tool workflow.

| Layer | Proposed choice if a wrapper is justified | Reason |
|---|---|---|
| Local UI | Python + Gradio, locally served assets, loopback-only | Audio input/playback and progress without a separate frontend build. |
| Application logic | Small typed Python package | Centralized validation, settings, profiles, and job orchestration. |
| Persistence | SQLite + application-owned filesystem | Durable local history without a service/database dependency. |
| Engine execution | Dedicated subprocess in a pinned environment | Contains dependency conflicts and makes cancellation/resource cleanup explicit. |
| Audio I/O | SoundFile and an explicitly pinned resampling implementation | Lossless input/output and controlled sample-rate conversion. |
| Diagnostics | Local structured JSON logs and manifests | Reproducibility without telemetry. |
| Optional later separation | Independently qualified local separator | Keep its artifacts and licensing separate from conversion. |

No Docker, Redis, Kubernetes, hosted authentication, remote queue, or separate REST service is required for the MVP.

```text
Local browser
    |
    v
Loopback UI + application service
    |              |
    |              +--> SQLite: profiles, assets, jobs, results
    |
    +--> Audio validation / preview selection / export
    |
    +--> Single-job supervisor
             |
             +--> Engine subprocess in pinned RVC environment
             |       --> pitch + content extraction
             |       --> target retrieval + synthesis
             |
             +--> Optional comparison engine in separate environment
             |
             +--> Guided training job, using the same exclusive GPU slot

All recordings, model files, caches, and outputs remain on local disk.
```

Only implement a second engine adapter if the comparison warrants it. The boundary should be narrow enough to replace an engine without inventing a general-purpose plugin platform.

## 2. Engine selection and environment qualification

### 2.1 Selected RVC configuration

Qualify an F0-enabled model with its exact sample rate, content encoder, vocoder, and retrieval index. Start with RMVPE pitch extraction, consistent with the Applio training guide.

Initial recipe to qualify:

- F0-enabled RVC target model with ContentVec features and a trained retrieval index.
- Standard HiFi-GAN/NSF path with a compatible pretrained generator/discriminator pair; in the inspected synthesizer, the standard F0-enabled HiFi-GAN selection instantiates `HiFiGANNSFGenerator`. Verify the actual class and metadata rather than relying on a UI label.
- Prefer a supported 48 kHz model/pretrained pair for the first recipe; native rate is a compatibility choice, not proof of superior sound.
- RMVPE extraction; zero semitone shift; autotune and proposed-pitch/median target-range adaptation off.
- High-register compatibility/folding off; assess extreme-register failures explicitly rather than changing octaves to hide them.
- No duration scaling or post-pitch effect.
- Unvoiced feature protection enabled and tuned on development material; no default aggressive denoising, gating, de-breathing, or post-pitch effects.

The pipeline passes a floating-point F0 trace to the generator in addition to coarse pitch features. The NSF source module uses that trace for excitation. Native segmentation slices from the padded full-track F0 sequence. This is the main source-level reason for selecting this controllable path, not a claim of sample-perfect synthesis.

In the inspected implementation, protection activates below `protect=0.5` and restores more pre-retrieval source features as its value decreases on unvoiced frames. Do not repeat the generic assumption that a higher slider value always means stronger protection. Freeze the behavior and chosen value for the qualified revision.

Use an officially supported pretrained generator/discriminator pair as the starting point for target-specific training. The inference model is not interchangeable with an arbitrary resumable training checkpoint. Import must distinguish the two.

Start with the documented default retrieval/consonant settings, then tune on development clips. Freeze final settings before qualification. Do not assume slider meanings remain identical across forks or revisions.

### 2.2 Reference-only candidates

#### SoulX-Singer-SVC: contingency, not initial implementation

Use the released `model-svc.pt` and `webui_svc.py` / SVC inference path, not the base lyrics/MIDI singing-synthesis path. In SoulX upstream terminology, the **prompt** is the desired voice reference and the **target audio** is the source song to convert. Those labels differ from this repository's target-voice terminology; map them explicitly in any wrapper.

The inspected SVC UI recommends a clean singing prompt and supports optional vocal separation and accompaniment mixing. Its automatic pitch shift is enabled by default: explicitly set `auto_shift=false` and `pitch_shift=0` for this product. Disable optional separation/mixing for the dry-vocal baseline. The SVC preprocessing path disables MIDI transcription.

Qualification must address:

- Requirements pin Torch 2.2.0; establish and freeze a coherent Blackwell-capable environment instead of assuming the published requirements work on the RTX 5070.
- Full pipeline VRAM is unknown. The 2.79 GB on-disk SVC checkpoint does not establish 12 GB peak-memory fit, especially with preprocessing models loaded.
- The upstream UI warns about identity instability on long/wide-range songs and trims prompt/source inputs to 30/600 seconds. Any integration must validate limits explicitly rather than silently truncate.
- The model quantizes F0 to 20-cent bins on a 20 ms analysis grid. This is a conditioning representation, not a measured error bound.
- Long-track segmentation filters regions with very little detected voicing and writes converted regions into an initially zero-valued waveform. Check isolated breaths/unvoiced spans explicitly; do not assume all zero-F0 regions are silence.
- Individual generated segments are cropped or padded to source length; equal file length does not by itself pass onset/alignment requirements.
- Upstream automatic accompaniment mixing downmixes and truncates to the shorter stem. It does not meet our stereo/timeline contract without adaptation; retain the dry output for quality evaluation.
- The inspected UI binds to all interfaces and can fall back to CPU. A qualified local workflow must enforce loopback-only access and explicit GPU failure behavior.
- Apache 2.0 is declared for the main code/weights; assess external preprocessing weights and dependencies separately.

These are concrete integration differences, not evidence that the model sounds better or worse than RVC. Run the same unchanged-pitch and breath-retention qualification before selecting it.

#### Seed-VC: optional legacy comparison

Use the v1 44.1 kHz F0-conditioned singing model, not the 22.05 kHz speech model or the v2 accent/style path.

Required semantic mapping from the upstream README:

| Proposed application property | Seed-VC setting |
|---|---|
| Preserve pitch | `f0-condition=True` |
| No target pitch matching | `auto-f0-adjust=False` |
| No transposition | `semi-tone-shift=0` |
| Preserve duration | `length-adjust=1.0` |
| Singing quality baseline | Compare 30 and 50 diffusion steps on development clips, then freeze one. |

The published minimum-reference and fine-tuning-speed claims are not product guarantees. The archived repository and old/conflicting dependencies make this a separately time-bounded experiment, not the recommended default.

### 2.3 Native Windows and Blackwell

Research found a documented RVC Windows/RTX 50-series installation path, but no local hardware execution was performed. Do not distribute an installer based only on documentation.

Qualification procedure:

1. Record Windows build, GPU model/VRAM, NVIDIA driver, RAM, free disk, and Python architecture.
2. Create an isolated environment for a selected engine commit. Follow a coherent upstream recipe; do not mix Applio's current Python/Torch requirements with an older RVC bundle.
3. Install matching Torch/audio packages with Blackwell-capable CUDA support. Verify actual resolved wheels, not merely the configured package index.
4. Run dependency consistency checks and import all pipeline modules.
5. Check CUDA availability and device capability, then execute a real GPU matrix operation and synchronize it.
6. Load the actual F0 extractor, content encoder, index, and generator and convert a 30-second dry vocal. This requires an actual authorized inference voice model: if none exists, do module/GPU checks first, then train the user's voice before this conversion. An explicitly authorized trusted starter voice is optional for an engine-only smoke check, never evidence of the user's likeness.
7. Run a short training/backpropagation job and save/reload its checkpoint before committing to a longer personal-model run.
8. Process full songs, inspect outputs, and record peak GPU and host memory.
9. Freeze the working lockfile, source commit, installer manifest, and model hashes.

The logical prerequisites above take precedence over list order: training supplies the first inference model when no authorized starter exists. The [Windows handoff](WINDOWS_HANDOFF.md) sequences this explicitly.

`torch.cuda.is_available()` alone is not success: incompatible kernels can still fail on first execution. The CUDA version shown by `nvidia-smi` reports driver capability, not the CUDA runtime in the installed PyTorch build.

Compatible prebuilt Torch wheels normally include their CUDA runtime dependencies. Installing a standalone CUDA Toolkit does not repair an incompatible Torch wheel. Add compiler/toolkit dependencies only if the selected engine actually requires building extensions.

Native Windows is the default. WSL2 is a fallback only after documenting a concrete native-Windows blocker and an explicit decision; it would require its own qualification and storage/install instructions.

## 3. Data model and storage

Store user data outside the source checkout, for example under `%LOCALAPPDATA%\LocalSingingVC`. This location is a proposal, configurable during setup. The handoff supplies a simpler existing-tool directory layout; the following database/profile layout is only for a justified future wrapper.

```text
LocalSingingVC/
  app.sqlite
  environments/
  engines/
  model-cache/
  profiles/<profile-id>/
    consent.json
    profile.json
    model/
    training-manifest.json
  projects/<project-id>/
    originals/
    derived/
    jobs/<job-id>/
      request.json
      events.jsonl
      staging/
      result.json
      outputs/
  diagnostics/
```

Do not commit audio, target checkpoints, indexes, credentials, model caches, or application databases to this repository.

### Core records

| Record | Important fields |
|---|---|
| `VoiceProfile` | ID, display name, consent status/scope/date, engine kind/revision, model/config/index/encoder hashes, native sample rate, F0 capability, qualified languages/styles/range, license references. |
| `AudioAsset` | ID, application-relative path, SHA-256, original filename for display, sample rate, channels, frames, format, validation findings. |
| `Job` | ID, kind, source asset, profile/version, preview range, settings snapshot, state, stage, progress, timestamps, cancellation request, worker identity, error code/message. |
| `Artifact` | ID, job ID, type, path, hash, sample rate/channels/frames, export gain, completion status. |
| `Environment` | ID, engine commit, dependency-lock hash, Python/Torch/CUDA versions, driver/GPU details, qualification result/date. |

Model hashes must cover the generator, index, encoder, pitch extractor, and relevant config, not just the target checkpoint. Profile edits create a new version; existing jobs retain their original snapshot.

Consent withdrawal prevents future conversions with the profile. It cannot recall audio already exported outside the application; communicate this limitation rather than implying technical enforcement of all downstream use.

## 4. End-to-end audio pipeline

### 4.1 Import and validate

- Copy an explicitly selected input into the project originals directory; never modify the source file.
- Validate file signature and decoded properties, not only the filename extension.
- Enforce file size, decoded duration/size, supported format/rate/channels, finite samples, and non-silence.
- Detect clipping and possible poor input quality as warnings. Do not certify single-singer isolation based on a heuristic.
- For stereo sources, require a channel/downmix choice and audition. Preserve that choice in the request.

P0 accepts dry vocals. It does not automatically run a separator when a mixed recording is imported. Explain the limitation and allow the user to supply a separate vocal stem.

### 4.2 Prepare model inputs

Maintain an immutable original and a canonical mono working signal plus exact source time coordinates. Resample separately for each model component's required rate; do not assume the content encoder and waveform generator share one input rate.

Do not trim pauses out of the source timeline. Training-only silence trimming is distinct from inference behavior.

Input scaling must be documented and reversible where applicable. Do not apply strong denoising, gating, reverb, or pitch processing by default.

### 4.3 Convert

RVC conceptual path:

```text
Source mono waveform
    +--> F0 extraction ------------------------------+
    +--> Content encoder --> Target feature retrieval|
                                                     v
                                        F0-conditioned generator
                                                     |
                                                     v
                                          Converted mono waveform
```

The content representation is not perfectly independent of speaker identity. Retrieval can reduce source leakage, but stronger retrieval can introduce artifacts. This is an empirical tuning trade-off.

Breaths and unvoiced consonants require a path through the acoustic representation and synthesis; an F0 track alone cannot encode them. Preserve their source timeline and evaluate the engine's protection settings on annotated examples. Do not discard low-F0-confidence frames as silence, copy the target reference's breath timing, or add a separate breath-replacement model in P0. Hard voicing masks and abrupt splices can create clicks and unnatural boundaries.

Run the model in evaluation/inference mode. Select numerical precision during qualification; do not automatically assume every model is safe in FP16. If a precision retry is offered, expose and record it.

### 4.4 Segment without losing time

Long-track handling must have **one owner per engine**. Prefer the engine's native long-audio implementation when it is bounded and passes continuity tests. Do not wrap it in a second unexamined splitting layer.

If application-owned segmentation is necessary:

- Start with nominal 15-second core intervals and additional context around boundaries; determine actual context/overlap from engine tests.
- Preserve original sample offsets for every interval, including silence.
- Use shared settings and target reference across chunks.
- Crop only known analysis/context padding and compensate only verified fixed algorithmic delay.
- Overlap-add with normalized window weights, with explicit treatment of the first and last interval.
- Check boundary notes, vibrato, breaths, and consonants for audible discontinuities.
- Ensure every source interval is covered once in the assembled timeline.

Keep a full-track pitch trace for diagnostics. Do not assume an upstream engine accepts externally supplied F0; that capability must be verified before implementing global-F0 conditioning.

Do not time-stretch, delete content, or append arbitrary silence to conceal an engine duration error. If unexpected timing drift exceeds the agreed tolerance, fail with `ALIGNMENT_FAILED` and retain diagnostics.

### 4.5 Export and optional mixing

Keep native-rate converted audio as an internal artifact. Resample once for the requested 44.1/48 kHz export and record that transformation.

Output duration must match the source timeline within the specified sample tolerance after legitimate padding/delay treatment. Separately evaluate lyric/note onsets: matching file length alone does not prove timing preservation.

For a supplied aligned backing stem, retain stereo, align sample coordinates, and mix only the converted lead with the backing. Do not mix the original lead back in. Warn that a backing stem created by imperfect separation may still contain the original singer.

Use explicit vocal/backing gains. If necessary, reduce final gain to avoid clipping and disclose the applied reduction; do not silently master, compress, or tune the performance. Export dry and mixed results separately.

Stage artifacts inside the destination filesystem, flush/close them, validate them, and atomically publish a completed result. A crash must never leave a partial file presented as a completed conversion.

## 5. Settings and adapter contract

Example request; IDs, paths, and defaults are illustrative, not an executable Applio request:

```json
{
  "schema_version": 1,
  "job_id": "generated-id",
  "kind": "convert",
  "source_asset_id": "asset-id",
  "voice_profile_id": "profile-id",
  "voice_profile_version": 1,
  "mode": "preserve_absolute_pitch",
  "preview": null,
  "settings": {
    "pitch_shift_semitones": 0,
    "auto_pitch_adjust": false,
    "autotune": false,
    "high_register_octave_fold": false,
    "duration_scale": 1.0,
    "post_pitch_shift": false,
    "export_sample_rate": 48000
  }
}
```

Resolve asset IDs to application-owned paths in the supervisor. Do not accept arbitrary filesystem paths or shell fragments from a browser request.

The adapter must:

- Declare supported model versions, input constraints, and semantic capabilities.
- Validate the profile bundle before job creation.
- Reject unknown or unsupported settings rather than ignore them.
- Map pitch/time semantics to the pinned upstream implementation and return the effective settings.
- Emit structured stage/progress events and actionable failure events.
- Return output metadata, settings, hashes, timings, and memory measurements.

Avoid using raw slider values as portable cross-engine semantics. RVC retrieval/protection controls and Seed-VC diffusion/guidance settings are engine-specific extensions, visible only when supported.

Proposed result statuses distinguish runtime completion from quality assessment:

```json
{
  "job_id": "generated-id",
  "status": "completed",
  "quality_status": "not_assessed",
  "effective_mode": "preserve_absolute_pitch",
  "artifacts": [],
  "warnings": [],
  "metrics": {
    "wall_seconds": null,
    "gpu_peak_allocated_bytes": null
  },
  "unavailable_reasons": {
    "wall_seconds": "Schema illustration, not a run.",
    "gpu_peak_allocated_bytes": "Schema illustration, not a run."
  },
  "environment_id": "qualified-environment-id"
}
```

The empty artifact list above is a schema illustration, **not a valid completed job**. Runtime validation requires at least one validated output artifact and real measured values where supported; unavailable metrics are explicitly null with a reason.

## 6. Job execution and failure behavior

For a future wrapper, use SQLite as the authoritative job-state store. A single supervisor acquires an application-instance lock and owns the exclusive GPU execution slot.

```text
queued -> validating -> running -> exporting -> completed
   |          |           |            |
   +----------+-----------+------------+--> failed
   +----------+-----------+------------+--> cancelled

On application restart:
  formerly active jobs -> interrupted
  queued jobs remain queued, awaiting explicit restart/resume action
```

At P0, launch a subprocess per job. This trades model-loading latency for simple resource reclamation and fault containment. Throughput measurements must include this overhead. Only add a resident GPU worker if measured loading time prevents the agreed performance goal.

Use argument arrays and a request-file protocol, never shell-concatenated user input. The child emits JSON Lines events; stdout/stderr remain available as bounded local diagnostics.

For cancellation, request cooperative stop at a safe boundary, then terminate only the job's owned process tree after a timeout. On Windows, use an owned Job Object or an equivalent process-tree containment mechanism. Do not kill processes by executable name.

Mark a job completed only after a successful worker exit, valid output metadata, published artifacts, and a committed database transaction. Reconcile filesystem/database state at startup.

### Explicit failure cases

| Code | Behavior |
|---|---|
| `INVALID_AUDIO` | Identify the violated input rule; do not submit a GPU job. |
| `MODEL_MISMATCH` | Report incompatible F0/version/sample-rate/encoder/index metadata. |
| `MODEL_UNTRUSTED` | Refuse loading an unapproved executable checkpoint or artifact. |
| `MISSING_ASSET` | List missing local assets; offer explicit setup/download outside offline inference. |
| `GPU_UNAVAILABLE` | Report device/driver/environment diagnosis; no CPU/cloud fallback. |
| `GPU_OOM` | Release resources and offer an explicit retry with a smaller supported chunk/batch setting. Record the changed setting. |
| `ALIGNMENT_FAILED` | Preserve diagnostics; do not disguise drift by stretching or cutting content. |
| `ENGINE_FAILED` | Surface a concise error plus local diagnostic details. |
| `DISK_FULL` | Preserve originals and completed outputs; mark current artifacts incomplete. |
| `INTERRUPTED` | Offer restart; do not claim arbitrary mid-track or mid-training resumption. |

Only resume training from an explicitly supported, intact checkpoint with matching recipe/configuration. Intermediate conversion chunks need not be resumable in P0.

## 7. Local training workflow

Training is target-specific adaptation using a pretrained RVC base, not foundation-model training.

The initial recording budget is 5-10 minutes, with 10 preferred. For a 10-minute total dataset, reserve separate one-minute development and evaluation takes, leaving roughly eight minutes for training before additional quality filtering. Record those actual usable durations. Do not imply this meets the guide's 10-30 minute training recommendation or guarantees a good model.

Use the development set for checkpoint/settings selection and hold the evaluation set back. If data sufficiency is unclear, compare a register-balanced subset of roughly four training minutes with the full available training set using the same development clips and documented training budgets. Collect additional missing registers/phonemes only when the results identify a gap. Repeating the same audio or training indefinitely is not a substitute for coverage.

Speech-only target recordings are technically usable for this family of workflows, but the preferred singing profile includes target singing across intended registers. Keep recording type in the dataset manifest and stratify evaluation accordingly. Do not require the same song as the source, and do not confuse the source's melody with the target data's purpose of teaching identity and acoustic coverage.

1. Create an enrollment profile and record consent/provenance.
2. Prepare and manually inspect the recordings.
3. Split by whole take/song/session into training, development, and held-out test sets before slicing.
4. Resample/slice training material using the pinned recipe, keeping a mapping back to original recordings. Inspect retained breaths.
5. Extract F0 and content features with the same encoder family expected at inference.
6. Begin with a conservative batch size and a short backpropagation/save/reload smoke check. Measure memory; change the configuration only through an explicit recorded update.
7. Save periodic checkpoints; inspect development conversion output and choose the checkpoint on development results.
8. Build the retrieval index from training features only.
9. Export the candidate inference bundle and run final tests without using the test set for tuning. Record its qualification status only after evaluation.

P0 can expose these steps through the upstream local training UI/CLI; a future import wizard is optional. The application should not schedule inference while guided training owns the GPU. If upstream training is launched outside a supervisor, finish it before running inference.

Record dataset hashes, source split membership, slice settings, seed, base-weight hashes, encoder/F0 versions, steps/epochs, batch size, checkpoint identity, and index parameters. Repeated runs may still vary across hardware/library versions; do not promise bitwise deterministic synthesis.

## 8. Privacy, security, and license boundaries

These are required operating constraints and proposed wrapper protections, not assertions that every upstream UI already implements them.

- Bind the UI to loopback only, disable public sharing, validate Host/Origin, and require per-launch local authorization for mutating/file-serving operations.
- Keep browser access scoped to application-owned artifact IDs; block path traversal and arbitrary directory browsing.
- Disable outbound analytics, remote TTS, auto-update checks, remote error reporting, and runtime asset downloads in offline mode. Cache browser assets locally.
- Download only pinned, approved model assets during explicit setup. Record hashes and licenses.
- Treat `.pth`/pickle checkpoints as potentially executable. Accept only locally generated or explicitly approved trusted-source weights; hashes establish identity, not inherent safety.
- Use restricted/weights-only loading where supported. Do not globally disable safe loading merely to make an old checkpoint load.
- Treat native index/parser files as untrusted too; validate provenance and load them only in the engine boundary.
- Run without administrator privileges. Ordinary subprocess isolation is not a security sandbox.
- Store recordings and voice models under user-only filesystem permissions. Recommend OS disk encryption; the MVP does not invent custom encryption.
- Confirm destructive deletion with an exact list and impact summary. Do not recursively delete application roots as an update mechanism.
- Treat licensing of code, weights, encoders, training data, codecs, and optional separators independently. Process boundaries do not settle copyleft questions.

The workflow must not upload samples to public demo sites for testing. Local user-controlled exports are distinct from automatically publishing or sharing recordings.

## 9. Verification plan and traceability

No tests in this section have been run; these are implementation requirements. The personal pilot does not need to implement a test framework or the proposed application to begin.

| Verification | Requirements covered | Specific check |
|---|---|---|
| Input contract tests | FR-01, FR-02 | Invalid formats, misleading extensions, oversized/long decoded files, NaN/Inf, silence, stereo decisions, immutable originals. |
| Settings contract tests | FR-05, FR-06 | Reject nonzero shifts, pitch matching, autotune, octave folding, time scaling, unknown options, and non-F0 profiles. |
| Adapter tests | FR-03, FR-14 | Exact profile compatibility and effective-setting mapping to the pinned engine; missing index behavior explicit. |
| Timeline tests | FR-06, FR-08, Q-03 | Impulses/onsets, long pauses, short final chunks, sustained notes crossing seams, sample-rate conversions, complete source coverage. |
| Job fault tests | FR-09, FR-12, NFR-05/06 | Cancel/kill/restart, OOM, disk-full simulation, malformed worker output, duplicate launch, atomic publication. |
| Local boundary tests | FR-11, NFR-04 | Loopback-only binding, unauthorized origins, path traversal, artifact access, no public share endpoint. |
| Offline integration | NFR-03, Q-12 | Block network and exercise launch, model load, conversion, guided training, and export with installed assets. |
| Hardware qualification | NFR-01/02, Q-09/10/11 | Real tensor execution, actual model path, short backpropagation, full songs, memory and latency recording. |
| Audio qualification | Q-01 through Q-07 | Independent F0 analysis, no time-warp/transposition, annotated onset checks, blind human listening. |
| Unvoiced-detail qualification | FR-15, Q-13 | Annotated breath timing/retention, unvoiced consonants, protection-setting trade-offs, no default breath removal, and audible source-identity leakage. |
| License/provenance review | NFR-08 | Complete inventory for exactly distributed code and assets, including optional dependencies. |

Synthetic tones, chirps, vibrato-like contours, silence, and impulses are pipeline tests, not evidence of convincing human voice conversion. Human singing clips remain necessary.

## 10. Observability and evaluation outputs

Every evaluated job writes:

- Source/target identifiers without unnecessary personal details.
- Hashes, engine revision, environment ID, and complete effective settings.
- Source and output frame counts/sample rates and any delay/padding treatment.
- Decode, model load, feature extraction, synthesis, assembly, and export timings where observable; unsupported measurements are null with reasons.
- Peak allocated/reserved GPU memory; total-device and host memory when obtainable.
- Errors, warnings, and any explicit retry/configuration change.

Keep evaluation outputs separate from ordinary job completion:

- Machine-readable per-case measurements.
- Blind-listening assignments and raw ratings stored locally if that optional study is run.
- Aggregate report split by target, source register, language, and typical/stress condition.
- Representative success and failure examples with participant permission.
- A decision record stating which gates passed, failed, or remain unknown.

No single embedding score or quality predictor should set `quality_status=passed`. That status belongs to a defined qualification protocol, not an inference heuristic. Personal acceptance is labeled separately from broader product qualification.

## 11. Phased implementation

### Phase A: Avoid premature application work

Establish the native Windows environment, collect one target dataset, and run the four-clip smoke experiment using existing tools. Train the user's model first if no authorized starter model is available. Measure actual behavior and retain all failing examples.

### Phase B: Select the engine on evidence

Train and freeze the selected RVC baseline on the available singing data. Use unchanged-pitch settings throughout. Perform a bounded recording/settings improvement if needed. Only if the result still misses agreed goals should SoulX-Singer-SVC become a comparative experiment; Seed-VC remains an optional legacy baseline, not a required implementation.

Vevo2 may be explored for noncommercial research if needed, but do not add it to a distributable product without resolving its checkpoint terms and hardware fit.

### Phase C: Adopt the existing app or fill workflow gaps

First assess whether the upstream application is sufficient. Where needed and explicitly justified, add profile import, audio validation, job supervision, preview/full conversion, persistence, and export around the existing engine. Add offline packaging and recovery behavior before optional features. Do not reimplement conversion algorithms already provided upstream.

### Phase D: Integrate enhancements only when justified

Integrated separation is the most likely next feature if users supply finished songs. Evaluate dry conversion, separated-vocal conversion, and final mix independently so separation problems are not mistaken for voice-model problems.

## 12. Main risks and responses

| Risk | Response |
|---|---|
| Target likeness poor despite clear audio | Inspect recording quality and register/phoneme coverage; perform one bounded settings/data iteration before a contingency comparison. Report limits. |
| Exact source notes sound unnatural in target voice | Keep the requirement; warn about range mismatch rather than transpose. |
| Applio/RVC dependency drift | Freeze a tested commit/environment and update deliberately. |
| Seed-VC archive/dependency problems | Bound the comparison effort; do not make the primary workflow depend on it prematurely. |
| Newer SoulX-Singer-SVC does not meet hardware/workflow requirements | Qualify memory and dependencies first; disable automatic pitch shift; retain explicit limits and unchanged-timeline export. Newer is not equivalent to production-ready. |
| Full-song OOM or seam artifacts | Single GPU job; qualify native chunking or implement one tested timeline-aware segmentation layer only if justified. |
| Accompaniment contains original singer | Accept clean stems first; document separation limitations when adding that feature. |
| Model licensing blocks distribution | Swap or omit the affected component before packaging; do not infer rights from a paper or code license. |
| Quality below goals | Stop custom-app expansion, retain the experiment, and revisit data/engine/scope explicitly. |

## 13. Repository handoff

These documents live under `docs/singing-voice-conversion/`. Start the Windows session with [WINDOWS_HANDOFF.md](WINDOWS_HANDOFF.md), preserving the source/target and pitch/time policies above.

Do not copy model binaries, voice recordings, or machine-specific environments into Git. The repository's [ignore rules](../../.gitignore) are only a backstop. Resolve the exact trusted asset acquisition process, engine commit, dependency lock, and real hashes on Windows; this documentation is not a tested installer or model manifest.
