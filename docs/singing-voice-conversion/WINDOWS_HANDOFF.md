# Windows RTX 5070 handoff

Status: **Unexecuted plan for a separate Windows Copilot CLI session.** This repository contains documentation only. No model installation, weights download, training, conversion, hardware benchmark, or perceptual evaluation was performed during the Mac handoff.

Read with: [Overview](../../README.md) -> this runbook -> [Specification](SPEC.md) -> [Research](RESEARCH.md) -> [Design](DESIGN.md).

## 1. Get this handoff onto the PC

The commands in this document are **unexecuted PowerShell examples**, compatible with Windows PowerShell 5.1 syntax. Run them one block at a time, inspect errors, and stop on failure. Do not paste them into Command Prompt or treat them as a tested installer.

The user is the sole developer and will continue on `main`. For a fresh checkout, start in a user-owned parent directory where `my-voice-singer` does not already exist:

```powershell
git clone --branch main --single-branch https://github.com/sanchar10/my-voice-singer.git
if ($LASTEXITCODE -ne 0) { throw "Clone failed; resolve access before continuing." }
Set-Location .\my-voice-singer
git status --short
Get-Content .\README.md
Get-Content .\docs\singing-voice-conversion\WINDOWS_HANDOFF.md
```

For an existing checkout, first inspect `git status --short` and preserve any work. Fetch `main`, switch to it, and fast-forward without resetting or overwriting local commits:

```powershell
git fetch origin refs/heads/main:refs/remotes/origin/main
if ($LASTEXITCODE -ne 0) { throw "Fetch failed." }
git switch main
if ($LASTEXITCODE -ne 0) { throw "Switch failed; inspect local branches and work before retrying." }
git merge --ff-only origin/main
if ($LASTEXITCODE -ne 0) { throw "Main has diverged; inspect commits before deciding how to reconcile them." }
```

The explicit fetch refspec also works for a previous single-branch handoff clone. If local `main` does not exist, `git switch main` normally creates it from `origin/main`; if remote selection is ambiguous, use `git switch --track origin/main` instead. Stop on divergence or uncommitted-work conflicts; do not force the fetch, reset, or discard work.

Open the Windows Copilot CLI session **in this checkout**, using the user's installed CLI. Inspect any repository instructions that exist by then; there are no application files or install scripts supplied by this handoff.

## 2. Copy/paste kickoff prompt

```text
Continue this local singing-voice experiment on my Windows RTX 5070 PC.
I am the sole developer and want to continue development on main.
Read README.md and all four docs in docs/singing-voice-conversion/
(WINDOWS_HANDOFF.md, SPEC.md, RESEARCH.md, DESIGN.md). Inspect repository
instructions, Git state, actual OS/GPU/VRAM/driver/RAM/disk, installed tools,
and existing environments before changing anything. Preserve unrelated work.

Goal: replace the ORIGINAL/source recording's singing identity with MY OWN
target voice, keeping absolute pitch at the corresponding time, octave,
melody, lyrics, phrasing, breath locations, and natural detail as far as the
model can achieve. My target training recordings need not be the source song.

Use ONE default: existing Applio UI with a target-specific F0-enabled RVC
model adapted from a compatible pretrained generator/discriminator pair,
ContentVec, RMVPE, standard F0-enabled HiFi-GAN/NSF path, and training-only
retrieval index. No custom app or model architecture now.

Follow current official upstream installation/docs, inspect the selected
revision, and resolve/pin a coherent Blackwell-capable isolated environment.
Never install requirements into global Python, blindly force upgrades, or
mix old bundles and newer requirements. Record exact revisions, wheels,
runtime, settings, trusted asset provenance, licenses, and actual hashes.
Native Windows first; WSL2 requires a demonstrated blocker and my decision.
No automatic CPU or cloud fallback, paid cloud work, or public audio uploads.
Keep UI loopback-only and sharing off. Keep all private assets outside Git.

First verify real CUDA tensor execution and required modules, not merely
cuda.is_available or nvidia-smi. Test the actual model path with a short dry
conversion only when an authorized inference voice model exists. If none
exists, do module/GPU checks, collect/split my recordings, and train my voice
before conversion. A trusted starter voice needs my explicit authorization
and is only an engine smoke check, not evidence of personal-voice quality.

Aim near 10 minutes of my clean dry varied singing; 5 minutes is exploratory.
Ten total minutes split about 8 train/1 development/1 held-out leaves only
8 training minutes, below Applio's 10-30 minute guidance, not a guaranteed
minimum. Split whole takes/songs before slicing; preserve originals/breaths,
inspect retained slices, and never put development/test data in the index.
Do not force uncomfortable notes or request more audio without a coverage gap.

Run short training/backpropagation and save/reload checks before a longer
bounded run. Tune checkpoint/retrieval/unvoiced protection on development
clips only. In inspected Applio, protect below 0.5 activates protection and
lower values restore more source features: verify that exact revision.
Keep pitch shift 0, autotune OFF, proposed/median target-range adaptation OFF,
high-register octave folding OFF, no duration scaling or post-pitch effect.
Record effective settings, not just UI labels.

Freeze settings; convert four short held-out sources covering sustained
notes, vibrato, breaths/pauses, and consonants/lyrics. Independently measure
absolute pitch and onset/timing, and listen dry for my likeness, breath
retention, naturalness, and failures. Equal duration is not pitch/time proof.
Then test a full song and optionally mix a separate aligned accompaniment.
Save reproducible local results and report pass/fail/unknown honestly.

This is a one-user pilot, not a requirement to recruit 3 singers/8 raters.
All SPEC numerical goals are proposed, not measured capabilities. After one
bounded RVC settings/data iteration, stop and discuss remaining failures.
Only then, or if enrollment becomes reference-only, consider SoulX-Singer-SVC;
archived Seed-VC is a legacy baseline, not the default. Do not expand scope,
conceal failures with transposition/time stretching, or claim studio-perfect
quality. End each milestone with evidence, remaining unknowns, and next action.
```

## 3. Inputs and private working layout

Obtain before the relevant milestone: the actual Windows/hardware inventory; a user-owned storage location; permission to download identified official assets; the user's recording device and intended languages/styles; personal versus distribution intent; clean target recordings; four authorized dry source clips and a full song; optionally an aligned backing stem. Do not block hardware/module checks while recordings are pending. If only mixed songs exist, document that limitation and decide on a separate local separation step before evaluating conversion.

Proposed root: `%LOCALAPPDATA%\LocalSingingVC`, or a user-chosen non-synced data drive with enough space. Verify permissions and whether the chosen location is cloud-synced. Do not store it inside this checkout. Paths below are conventions for the experiment, **not upstream-required paths**:

```text
LocalSingingVC/
  engines/applio/                 upstream checkout or official installation
  environments/                  if not managed inside the upstream installation
  assets/                        approved base pair, ContentVec, RMVPE, license copies
  recordings/originals/          immutable user takes
  datasets/personal-v1/          split manifest, derived training slices/features
  sources/originals/             authorized dry sources and optional backing
  voices/personal-v1/            training checkpoints, inference model, training-only index
  experiments/<run-id>/          settings, logs, environment, measurements, outputs
  diagnostics/                   hardware/module checks, no secrets
```

Use the upstream tool's expected internal paths when necessary; the upstream install is itself outside the docs repo. Keep manifests mapping those paths to logical asset IDs. Do not move checkpoints independently of their configuration/index metadata. Back up originals privately before preprocessing.

### Recording plan

Aim for about 10 usable minutes; do not demand 30 before a first pilot. Suggested capture: five minutes varied sung phrases, two sustained vowels/comfortable scales, one comfortable register/technique coverage, and two separate development/evaluation takes. This is a flexible plan, not a measured optimal recipe.

Record lossless mono WAV/FLAC at a supported rate, preferably 44.1/48 kHz. One singer, dry room, stable mic position, no accompaniment, reverb, autotune, doubled vocals, clipping, or heavy denoising. Include comfortable low/mid/high registers, different vowels, lyrics, consonants, natural breaths, and intended techniques. Never strain to reach the source singer's range.

Split **whole takes/songs before slicing**, keeping related takes together where leakage is possible. Aim near 8/1/1 minutes for training/development/held-out material, then record actual usable durations after filtering. Eight training minutes remains below Applio's 10-30 minute recommendation; the recommendation is neither a hard minimum nor a quality guarantee. For five total minutes, reserve development/test material too and label the smaller training set exploratory. Do not duplicate clips to increase duration.

The target held-out takes are identity references, not index/training data. Source evaluation clips must also be absent from training and development; they need not be sung by the user. Select source clips from authorized material covering the four cases below. No need for the target to record the source song. More recordings are justified only by identified missing coverage.

## 4. Milestones and exit checks

Complete in order. A successful import or tensor test is not a conversion, and a successful conversion is not a quality pass.

### M0: Inventory and safe acquisition

- [ ] Read repo instructions/status; confirm scope and storage. Record OS build, GPU/VRAM, driver, RAM, free space, tools, and existing environments. Research planning provisions of 32 GB RAM/50 GB free SSD are not measured hard requirements.
- [ ] Review the current [Applio installation guide](https://docs.applio.org/getting-started/installation/), [official repository](https://github.com/IAHispano/Applio), [Windows installer](https://github.com/IAHispano/Applio/blob/main/run-install.bat), and [requirements](https://github.com/IAHispano/Applio/blob/main/requirements.txt) together. Resolve any script/docs discrepancy for one selected revision; do not splice recipes.
- [ ] Review [training](https://docs.applio.org/getting-started/training/), [pretrained models](https://docs.applio.org/getting-started/pretrained/), and [inference](https://docs.applio.org/getting-started/inference/). Acquire only authorized assets from their documented official sources.
- [ ] Record exact commit/release, installer identity, asset URLs/revisions, actual SHA-256 values, and separate code/weight/dependency licenses. Research Git blob IDs are historical evidence, not runtime pins or model hashes.

**Exit:** inventory and acquisition plan saved locally. No arbitrary marketplace model, demo upload, administrator-level model execution, public UI link, or disabling safe checkpoint loading to bypass errors.

Example read-only inventory; not executed here:

```powershell
$ErrorActionPreference = "Stop"
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber, OSArchitecture
Get-CimInstance Win32_ComputerSystem | Select-Object TotalPhysicalMemory
Get-PSDrive -PSProvider FileSystem | Select-Object Name, Root, Used, Free
nvidia-smi
if ($LASTEXITCODE -ne 0) { throw "NVIDIA inventory failed; diagnose driver/device before setup." }
```

`nvidia-smi` reports driver-supported CUDA capability, not the CUDA runtime packaged with Torch. Do not infer the installed wheel or all kernel support from its banner.

### M1: Isolated installation and real GPU smoke check

- [ ] Follow the inspected official installation flow outside this repo, in its dedicated environment. Confirm which Python executable the launcher actually uses. Do not install requirements into a global Python or blindly upgrade packages.
- [ ] Resolve current compatible Python/Torch/torchaudio/CUDA wheels for the selected Applio revision and RTX 5070. The historical version examples in research are not install commands or a lockfile. Prebuilt wheels normally supply runtime dependencies; install a full CUDA Toolkit only if a demonstrated extension-build requirement needs it.
- [ ] Run dependency consistency checks, capture the resolved packages, import the actual pipeline modules, and test CUDA kernels. Record the environment before changing it.
- [ ] Launch the existing UI on loopback only, sharing disabled, and verify its listener and local response. Keep one GPU job active at a time.
- [ ] If an authorized compatible inference voice exists, run a roughly 30-second dry source through RMVPE, ContentVec, index retrieval, and the F0-enabled generator on GPU with zero shift. Inspect the waveform, logs, and actual device usage. Otherwise mark model-path conversion **pending M3**, not passed.

For a kernel check **after installation**, choose the actual environment interpreter. This block installs nothing and is not a substitute for module/model checks:

```powershell
$EnginePython = Read-Host "Full path to the selected Applio environment's python.exe"
if (-not (Test-Path -LiteralPath $EnginePython -PathType Leaf)) { throw "Interpreter not found." }
& $EnginePython -m pip check
if ($LASTEXITCODE -ne 0) { throw "Dependency consistency check failed." }
@'
import sys
import torch

print("python", sys.executable, sys.version)
print("torch", torch.__version__, "torch_cuda_runtime", torch.version.cuda)
if not torch.cuda.is_available():
    raise RuntimeError("CUDA unavailable; no CPU fallback permitted.")
print("gpu", torch.cuda.get_device_name(0))
print("capability", torch.cuda.get_device_capability(0))
print("compiled_architectures", torch.cuda.get_arch_list())
x = torch.randn(512, 512, device="cuda", requires_grad=True)
y = x @ x.T
loss = y.square().mean()
loss.backward()
torch.cuda.synchronize()
if not torch.isfinite(y).all().item() or not torch.isfinite(x.grad).all().item():
    raise RuntimeError("Non-finite GPU result.")
print("GPU matrix operation and tensor backpropagation completed on", y.device)
'@ | & $EnginePython -
if ($LASTEXITCODE -ne 0) { throw "GPU kernel check failed; stop before model work." }
```

This generic tensor backpropagation is **not** the RVC training smoke check. `pip check` also cannot prove native-extension compatibility. Load and exercise the actual chosen model components; do not fabricate import/CLI flags from the proposed DESIGN interfaces. If the official environment has no pip, use its documented dependency consistency/export mechanism and record that difference rather than installing into global Python.

**Exit:** working isolated modules/kernels and explicit model-path status. An authorized starter model proves only engine execution, not personal voice likeness. Stop on incompatibility; no automatic CPU/cloud retry.

### M2: Collect, split, and inspect target data

- [ ] Capture the user's 5-10 minutes following the recording plan. Save consent/use scope and immutable originals locally.
- [ ] Assign whole takes/songs to train/development/test first. Record hashes, duration, register/language coverage, and grouping. Check no overlap or adjacent-slice leakage.
- [ ] Preprocess training copies only with the pinned upstream recipe. Inspect slices for missing breaths, truncated consonants, silence, clipping, and poor F0 coverage. Do not silently remove every unvoiced interval.

**Exit:** actual usable minutes per split and a reviewed training-only dataset. Missing coverage is a recorded limitation, not a reason to force uncomfortable singing or demand more data prematurely.

### M3: Train the personal voice and build the index

- [ ] Verify compatible **pretrained generator/discriminator pair**, sample rate, model version, ContentVec dimensions, F0 capability, and standard HiFi-GAN/NSF path. Prefer a supported 48 kHz pair if coherent with upstream; do not select rate solely for perceived quality.
- [ ] Extract training F0 with RMVPE and ContentVec features. Verify actual GPU execution where expected.
- [ ] Start a small supported batch, run a short **actual RVC training/backpropagation** check, save/reload a compatible checkpoint, and record memory/errors before a longer run.
- [ ] Agree a finite run/checkpoint budget based on observed progress; no invented epoch/time guarantee. Train from the base pair, not from random initialization. Save configuration, seed, actual steps/epochs, batches, and checkpoint IDs.
- [ ] Build the retrieval index from **training features only**. Export the correct inference model; a resumable training checkpoint is not automatically an inference model.
- [ ] Convert development clips to select checkpoint, retrieval strength, and protection. Verify the pinned code: protection below `0.5` activates blending; lower restores more source features on unvoiced frames. Naturalness may trade off against source identity leakage.
- [ ] Complete the real short dry model-path smoke check if deferred from M1. Freeze settings and hashes before held-out evaluation.

**Exit:** a reloadable personal inference model/index bundle and successful short conversion on the actual GPU. Do not call the model qualified just because training finished.

### M4: Four held-out conversions at unchanged pitch

Use four 10-30 second **dry** sources, none used to choose settings/checkpoints:

| Case | Include and inspect |
|---|---|
| Sustained note | Stable vowel, absolute note/octave, onset, steady target identity. |
| Vibrato/transition | Natural vibrato and comfortable register transition; check contour/time and instability. |
| Breath/pause | Clearly audible inhalation, aspiration, unvoiced span and pause; check retained location/texture, no gating or added breath. |
| Consonants/lyrics | Varied consonants and sung words; check articulation, lyric/onset preservation, source leakage. |

Before each job, record effective settings: **pitch shift 0; autotune off; automatic/proposed/median target-range adaptation off; high-register octave-fold compatibility off; no duration scaling or post-pitch effect**. Confirm actual flag mappings against the pinned code and save all other settings, including retrieval, protection, precision, segmentation, resampling, and gain.

Run independent F0 analysis (for example a separately resolved/pinned pYIN analysis environment), not just RMVPE's conditioning trace. Compare absolute pitch at original source time; do not octave-normalize, transpose, dynamically time-warp, or crop content to improve scores. Inspect tracker uncertainty manually. Report voiced coverage/dropouts, not only median error or correlation.

Annotate source note/lyric and breath onsets before conversion. Compare onset timing and whole-file duration separately, allowing only verified fixed algorithmic delay handling. Listen dry at matched playback loudness against the source and held-out **user identity references**. Score likeness, naturalness, lyrics, breath retention, and artifacts separately; reference loudness matching must not alter exports.

**Exit:** all four cases have outputs or explicit errors, measurements or honest unknowns, and personal listening decisions. Freeze and retain failures; do not remove them from the denominator.

### M5: Full song, optional mix, and repeatability

- [ ] With frozen settings, process one authorized full dry song. Check complete timeline, sustained notes across chunk seams, pauses, breath retention, pitch/octave, memory, and wall time. Do not wrap native chunking in a second untested splitter.
- [ ] Reopen/reload the saved bundle and repeat a short conversion. Capture effective settings and compare behavior; do not promise bitwise deterministic output.
- [ ] After caching required assets, verify local launch/conversion/export with networking blocked. Test training offline only if claiming that part works offline; unknown is not passed.
- [ ] Optionally mix a separate aligned backing stem, preserving stereo and explicit gains. Keep dry output. Never mix the original lead back in; disclose residual source vocals from imperfect separation. Do not truncate to the shorter stem or stretch vocals to conceal drift.
- [ ] Save local manifest, environment snapshot, outputs, diagnostics, metrics, listening notes, and a decision record. Review any sanitized report before committing it; do not publish audio or sensitive manifests.

**Exit:** a reproducible personal-use result with explicit limitations, or a documented failure. Optional broader qualification requires its own corpus/panel; this milestone does not establish it.

## 5. Pass/fail and bounded decisions

| Gate | Pass evidence | Failure or unknown behavior |
|---|---|---|
| Environment smoke | Correct isolated interpreter; dependency/module checks, real GPU operation, actual model path, short RVC training save/reload. | Missing voice model means pending, not passed. Kernel/OOM/model mismatch stops the affected milestone; diagnose without hidden fallback. |
| Personal quality pilot | All four cases assessed; user finds their identity recognizable/useful; independent pitch/time and breath findings recorded against predeclared targets. | Runtime success alone is insufficient. Any octave shift, deleted breath/content, mangled lyrics, or unacceptable likeness is recorded as a failure; uncertain analysis remains unknown. |
| Full-song usability | Complete timeline and dry output; chunk continuity, observed memory/speed, repeat load and applicable offline checks recorded. | Do not hide drift with time stretching/padding, artifacts with accompaniment, or missing measurements with invented values. |
| Broader product qualification | The separate SPEC protocol and applicable gates actually executed. | Explicitly **not performed** for the ordinary one-user pilot. |

For the pilot, use SPEC Q-01/Q-02/Q-03/Q-13 as **proposed diagnostic targets**: median absolute pitch error at most 25 cents; at least 90% of reference-voiced frames within 50 cents; voicing disagreement at most 5%; duration within one export sample; onset error 95th percentile at most 50 ms; at least 90% annotated breaths retained with onset deviation at most 100 ms. Missing/unvoiced output frames count against reference coverage. Report counts because four clips may contain few breath/onset events. Human majority-rating criteria cannot be claimed from a single listener.

Record pass/fail/unknown per metric; personally useful output may still miss a proposed product target. If accepting that result, label the explicit personal-use limitation, not "SPEC passed." Freeze tolerances before final evaluation. SPEC's speed targets are unmeasured proposals, not guarantees or reasons to change pitch.

Allow **one bounded RVC improvement round** after the baseline: identify a specific cause, predeclare a small development settings/checkpoint comparison or targeted recording gap, keep the source-pitch policy unchanged, and stop at the agreed budget. If held-out failures inform new tuning, that set is no longer blind; retain the first result and label the rerun as reused-test evidence, or reserve a fresh whole-take test for a new claim.

Only if quality still misses agreed goals, or enrollment changes to reference-only, discuss SoulX-Singer-SVC as a separate, bounded environment experiment. Its source-level pitch defaults, F0 quantization, breath-segmentation behavior, old Torch pin, unknown VRAM, and source-length padding must be reviewed in [RESEARCH.md](RESEARCH.md) and [DESIGN.md](DESIGN.md). Archived Seed-VC remains a legacy baseline. A custom UI is considered only after useful quality is demonstrated and a specific upstream workflow gap is recorded.

## 6. Local experiment log and result manifest

Keep these **outside Git** by default, one directory per run. Templates below are proposed record formats, not upstream APIs or measured results. Replace nulls only with observed facts; use explicit unavailable reasons. A dependency export is a snapshot, not automatically a reproducible lock: retain installer/revision, wheel sources/versions, model hashes, and the selected resolver's lock/export when available.

Suggested local log header:

```text
Run ID / timestamp:
Purpose: environment smoke | development | held-out pilot | full song
Repo docs commit / Applio revision:
Hypothesis / allowed changes / budget:
Inputs and consent/provenance references:
Dataset version / usable train-dev-test durations / whole-take split:
Environment and asset manifest paths:
Effective settings and model/index bundle:
Steps executed / exact commands or UI actions:
Outputs / errors / explicit retries:
Pitch and timing findings:
Target likeness / naturalness / lyrics / breath findings:
Decision: pass | fail | unknown; scope and reason:
Next action / stop condition:
```

Proposed `result.json` skeleton:

```json
{
  "schema_version": 1,
  "run_id": null,
  "scope": "personal_pilot",
  "runtime_status": "not_run",
  "quality_status": "not_assessed",
  "docs_commit": null,
  "environment": {
    "engine_commit": null,
    "installer_sha256": null,
    "os_build": null,
    "gpu_name": null,
    "gpu_vram_bytes": null,
    "driver_version": null,
    "python_version": null,
    "torch_version": null,
    "torch_cuda_runtime": null,
    "dependency_snapshot_path": null,
    "lock_path": null
  },
  "dataset": {
    "manifest_path": null,
    "split_unit": "whole_take_or_song",
    "usable_train_seconds": null,
    "usable_development_seconds": null,
    "usable_test_seconds": null,
    "index_training_only_verified": null
  },
  "assets": [
    {
      "role": "inference_generator",
      "local_path": null,
      "source_url_or_training_run": null,
      "revision": null,
      "sha256": null,
      "license_reference": null,
      "authorized": null
    }
  ],
  "training": {
    "config_path": null,
    "seed": null,
    "epochs": null,
    "steps": null,
    "batch_size": null,
    "selected_checkpoint": null,
    "index_parameters_path": null
  },
  "settings": {
    "mode": "preserve_absolute_pitch",
    "pitch_shift_semitones": 0,
    "autotune": false,
    "auto_or_proposed_pitch_adaptation": false,
    "high_register_octave_fold": false,
    "duration_scale": 1.0,
    "post_pitch_shift": false,
    "f0_extractor": "RMVPE",
    "content_encoder": "ContentVec",
    "vocoder_path": "F0-enabled HiFi-GAN/NSF",
    "retrieval_strength": null,
    "protect": null,
    "precision": null,
    "native_sample_rate": null,
    "effective_upstream_settings_path": null
  },
  "case": {
    "source_id": null,
    "source_sha256": null,
    "source_start_seconds": null,
    "source_sample_rate": null,
    "source_frames": null,
    "output_path": null,
    "output_sha256": null,
    "output_sample_rate": null,
    "output_frames": null,
    "delay_padding_resampling_gain_notes": null
  },
  "measurements": {
    "analysis_tracker_and_version": null,
    "pitch_median_absolute_cents": null,
    "reference_voiced_frames": null,
    "reference_frames_within_50_cents": null,
    "voicing_disagreement_percent": null,
    "duration_error_export_samples": null,
    "onset_error_p95_ms": null,
    "annotated_breath_count": null,
    "retained_breath_count": null,
    "breath_onset_errors_ms": null,
    "wall_seconds": null,
    "gpu_peak_allocated_bytes": null,
    "gpu_peak_reserved_bytes": null,
    "total_device_peak_bytes": null,
    "host_peak_bytes": null
  },
  "listening": {
    "protocol": "personal_nonblinded",
    "target_reference_ids": [],
    "target_likeness": null,
    "naturalness": null,
    "lyrics_and_consonants": null,
    "breath_texture_and_source_leakage": null,
    "major_artifacts": null
  },
  "gate_decisions": [],
  "errors_and_retries": [],
  "unavailable_reasons": {
    "all_observations": "Template only; no experiment has run."
  }
}
```

Create one case record per source, including failed cases. Expand `assets` for base generator/discriminator, trained generator, retrieval index, ContentVec, RMVPE, configurations, source audio and held-out references; private dataset manifests hold take hashes and permissions. Record actual full upstream settings, preprocessing/slicing, segmentation, precision, and analysis settings in linked local files. Never invent a hash or license. No `completed` state without a validated output; no quality pass without the named evaluation.

Example snapshot/hash commands, **unexecuted**, after selecting `$EnginePython` in M1 and creating an experiment directory:

```powershell
$RunDirectory = Read-Host "Full path to an existing private experiment directory outside the repo"
if (-not (Test-Path -LiteralPath $RunDirectory -PathType Container)) { throw "Run directory not found." }
$SnapshotPath = Join-Path $RunDirectory "packages-freeze.txt"
if (Test-Path -LiteralPath $SnapshotPath) { throw "Snapshot already exists; use a new run directory." }
$Packages = & $EnginePython -m pip freeze --all
if ($LASTEXITCODE -ne 0) { throw "Package snapshot failed." }
$Packages | Set-Content -LiteralPath $SnapshotPath -Encoding UTF8
$AssetPath = Read-Host "Full path to one approved asset to hash"
Get-FileHash -LiteralPath $AssetPath -Algorithm SHA256
```

Review package snapshots for local usernames, private URLs, or credentials before sharing. Do not automatically commit generated manifests: the ignore rules deliberately leave reviewed text manifests trackable, so they are not a privacy enforcement mechanism.

## 7. Troubleshooting and hard stops

| Symptom | Next diagnostic/action | Do not |
|---|---|---|
| CUDA unavailable, unsupported kernel, or unexpected CPU execution | Check the exact launcher interpreter, resolved Torch runtime/wheels, driver/device capability, dependency coherence, and actual failing operation. Rebuild only the isolated environment from a coherent approved recipe if needed. | Infer runtime from `nvidia-smi`, force-upgrade global Python, automatically use CPU/cloud, or install a toolkit as a generic cure. |
| Missing model/index or sample-rate/F0/encoder mismatch | Inspect compatible bundle metadata and approved asset provenance. Train the user model if none exists. | Treat an arbitrary checkpoint as an inference model or quietly skip the index for the qualified recipe. |
| OOM | Record full error and memory/device activity; finish the session's other GPU job; retry an explicitly recorded smaller supported batch/chunk or qualified precision. | Kill unrelated processes, silently reduce quality, or claim 12 GB fit from checkpoint disk size. |
| Octave error or pitch drift | Check source F0, independent tracker, high-register folding, hidden adaptation, and effective settings. Keep the original notes and retain the failure. | Transpose, octave-normalize, autotune, or time-warp to pass. |
| Missing/metallic breaths or consonants | Inspect original/slices, unvoiced features, retrieval/protection direction, and segment boundaries on development material. | Hard-gate unvoiced spans or paste the target reference's breath timing into the source. |
| Weak personal likeness | Confirm model/index identity, recording quality, register/phoneme coverage, and held-out identity references; use the bounded development iteration. | Increase training indefinitely, demand unrelated recordings, or use accompaniment to hide the result. |
| Full-song drift or seams | Inspect native segmentation, exact offsets, pause coverage, rate conversion, and verified delay. | Add a second blind splitter, pad/cut content to fake alignment, or equate equal duration with preserved onsets. |
| Native Windows install blocked | Save exact command/error and upstream revision; attempt a coherent native fix. Discuss WSL2 only for a demonstrated blocker. | Migrate to WSL2 automatically or mix environments. |
| Missing permission, untrusted download, unsafe loader workaround, or unexplained outbound traffic | Stop affected acquisition/execution, identify the source/license/network behavior, and get an explicit decision. | Upload private audio, bypass safe loading globally, expose the UI publicly, or proceed with unclear rights. |

Also stop if originals would be overwritten, output storage is insufficient, the agreed training/iteration budget is exhausted, or the user is asked to sing uncomfortable notes. Preserve local evidence and report the exact blocker rather than claiming completion.

At the handoff's end, report the environment and model identity, local result location, what actually ran, personal pass/fail/unknown decisions, and any next bounded action. Publish only deliberately reviewed documentation; keep personal audio, weights, credentials, and raw results local.
