# Local Singing Voice Conversion: Product Specification

Version: 0.3 draft, repository handoff

Date: 2026-09-25, Pacific time

Status: Proposed scope and acceptance criteria; not an implemented or validated product.

Companion documents: [Windows handoff](WINDOWS_HANDOFF.md) | [Research and evidence](RESEARCH.md) | [Technical design](DESIGN.md) | [Repository overview](../../README.md)

## 1. Product goal

Let the user import a solo singing recording and replace the original singer's identity with **the user's own voice**, without intentionally changing the source's absolute notes, octave, melody, lyrics, timing, phrasing, or breath locations. Preserve natural vocal detail as far as the model can achieve.

**Source/original** is the performance being converted. **Target voice** is the user's identity learned from their own recordings. Broader references to consenting target singers below describe possible later qualification, not a requirement to enroll anyone else for the personal pilot.

The complete core workflow is intended to run on the user's Windows PC with an RTX 5070. There are no required cloud services or recurring API fees.

Success means a useful, repeatable creative tool, not a guarantee of impersonation-level indistinguishability or a simulation of the target person's singing skills.

Selected solution: **Applio with one target-specific, F0-enabled RVC model adapted from compatible pretrained weights.** Use the existing application first. A second engine is a contingency, not required initial scope.

## 2. Confirmed needs and provisional assumptions

| Category | Requirement or assumption |
|---|---|
| Confirmed | Desktop Windows PC with an RTX 5070, expected 12 GB; no paid cloud processing. Verify the actual hardware in the Windows session. |
| Confirmed | Singing-to-singing conversion into the user's own voice while keeping the original pitch/tune, including octave. |
| Confirmed | Match the target vocal tone, preserve pitch at the corresponding time, and handle breaths and other vocal detail naturally. Prefer existing end-to-end open-source software over rebuilding the pipeline. |
| Confirmed | This repository holds the research, specification, design, and Windows handoff. Actual setup and experiments belong to a separate CLI session on the user's PC, not this Mac documentation session. |
| Confirmed | The user is willing to record 5-10 minutes of target singing for improved quality or training. |
| Proposed MVP | Single operator on one PC; offline file processing, not live microphone conversion. |
| Proposed MVP | Dry single-singer source vocals; optional user-supplied aligned accompaniment. |
| Selected enrollment | Train a reusable RVC target voice in Applio. Prefer the upper end of the 5-10 minute initial recording budget. |
| Unconfirmed | Exact Windows version, RAM, free disk, intended languages/genres, actual quality and range coverage of the recordings. |
| Unconfirmed | Personal-only versus eventual commercial distribution. |

These assumptions are reversible design proposals. Confirm the unconfirmed items before implementation; do not silently reinterpret the pitch requirement to improve a demo.

## 3. Users and primary workflow

Primary user: a creator converting authorized source recordings into their own voice.

1. Launch the existing local app and review its GPU/environment readiness.
2. Train/enroll the user's target voice or select an existing, qualified personal voice model.
3. Import a dry singing track and audition the source.
4. Select a 10-30 second preview segment.
5. Convert with **Preserve original pitch** enabled and locked.
6. Compare source and converted audio at matched playback loudness.
7. Convert the full track after approving the preview.
8. Export a dry converted vocal and, if supplied, an aligned accompaniment mix.

Playback loudness matching is for comparison only. It must not silently modify the exported master. The named mode is a proposed semantic requirement, not a claim that the upstream UI has this exact label or locking behavior; verify its actual controls.

## 4. Scope

The following P0/P1 requirements describe a possible usable workflow/product. They are not features already implemented in this repository or a mandate to build a custom app before the personal pilot. Record upstream gaps first; adopting the existing UI is a valid final outcome.

### P0: Required for the first usable version

- Local file import, audio validation, and source playback.
- Windows/GPU readiness checks that actually exercise the selected engine.
- Target voice profile registration with consent and model provenance.
- Guided local RVC training using the pinned upstream workflow, followed by model/index import. A custom training UI is not required at P0.
- One qualified RVC-based inference engine.
- Zero-transposition singing conversion, previews, full tracks, job status, cancellation, explicit errors, and local history.
- Dry WAV export; optional mixing with a separately supplied, already aligned backing track.
- Local data storage, deletion controls, reproducibility manifests, and offline operation after setup.
- A documented quality evaluation on held-out recordings.

### P1: Only after the core quality gate

- Integrated dataset preparation and training UI.
- Reference-only adapter only if RVC does not meet the agreed needs after bounded tuning or the enrollment requirement changes; SoulX-Singer-SVC is the contingency candidate, not a second required engine.
- Integrated vocal separation/de-reverb, with a separately qualified model and license.
- Batch conversion, multiple target profiles, recording through the UI, and richer mixing.
- Fine-grained pitch visualization and diagnostic tools.

### Explicit non-goals

- Real-time voice changing, mobile deployment, cloud hosting, shared accounts, or remote access.
- Arbitrary public figure/marketplace voice downloads.
- Lyrics generation, translation, text-to-singing, accompaniment generation, or melody correction.
- Automatic key changes, octave shifts, autotune, or target-range adaptation.
- Separating individual singers from a choir or converting harmonies in one pass.
- Training a general-purpose foundation voice model from scratch.
- Promising quality for every language, genre, vocal register, or recording condition.

## 5. Functional requirements

| ID | Requirement | Acceptance behavior |
|---|---|---|
| FR-01 | Import local audio. | Accept supported WAV/FLAC files; reject unsupported, malformed, empty, non-finite, or over-limit files with specific messages. |
| FR-02 | Preserve original media. | Originals are immutable; processing writes distinct derived files. |
| FR-03 | Register target voices. | Store consent scope, origin, engine, model/config/index compatibility, hashes, and qualification status. Unqualified voices are visibly experimental. |
| FR-04 | Support local target preparation. | Provide a versioned recording/training recipe, held-out validation split, training manifest, and compatible inference-model export/import. |
| FR-05 | Preserve pitch. | Use an F0-enabled singing model, zero semitone shift, no automatic pitch matching, and no pitch-correction effects. Conflicting settings are rejected. |
| FR-06 | Preserve time. | Keep source starts, pauses, lyrics timing, and duration; do not concatenate only detected singing regions. |
| FR-07 | Preview conversion. | User selects a 10-30 second region; audition uses its original timeline position and records the settings used. |
| FR-08 | Convert a full track. | Process within the documented limits using the same voice/model and semantic settings as the preview. Chunk-related differences must be evaluated. |
| FR-09 | Manage jobs. | Show queued/running/completed/failed/cancelled/interrupted states, stage progress, cancellation, and actionable errors. |
| FR-10 | Export results. | Export a dry WAV and JSON provenance manifest; optional mix is a separate artifact. Never overwrite the source. |
| FR-11 | Preserve local privacy. | Core inference works with outbound networking blocked after setup; no public UI sharing link or telemetry. |
| FR-12 | Diagnose failures. | Distinguish missing weights, incompatible environment, GPU OOM, invalid audio, unsafe model import, and alignment failure. No hidden CPU/cloud fallback. |
| FR-13 | Manage stored data. | Delete voice data or project artifacts only with a preview of affected items and explicit confirmation. Disallow new jobs for a revoked profile. |
| FR-14 | Record reproducibility. | Every output records engine revision, model hashes, settings, input hash, sample rates, timings, and environment identity. |
| FR-15 | Preserve non-pitched vocal detail. | Retain breath events and unvoiced consonant timing; avoid default gating/de-breathing. Evaluate naturalness and source-identity leakage separately from F0 accuracy. |

### Proposed input and output contract

These are product limits chosen for an MVP, not model limits inferred from documentation.

| Item | P0 contract |
|---|---|
| Source format | WAV PCM 16/24-bit or finite float32; FLAC 16/24-bit. |
| Source rate | 16-96 kHz, decoded and resampled to the selected engine's required rate. Recommend recording at 44.1 or 48 kHz. |
| Source channels | Mono preferred. Stereo requires an explicit left/right/downmix selection and audition; no unannounced channel averaging. |
| Source length | 1 second to 10 minutes. |
| Uploaded file size | Maximum 250 MiB per source/backing file; also enforce decoded-size and duration limits. |
| Target training material | Initial user budget: 5-10 minutes of singing, preferably 10. Reserve development/test takes. Published Applio guidance is 10-30 minutes; this smaller-data pilot is not a quality guarantee. |
| Preview | 10-30 seconds where source length permits; shorter clips preview in full. |
| Dry export | Mono 24-bit PCM WAV, user chooses 44.1 or 48 kHz; default 48 kHz. Record native model rate separately. |
| Backing input | Optional mono/stereo WAV/FLAC on the same known timeline; mismatched duration requires correction outside P0. |
| Mix export | Stereo 24-bit PCM WAV at the chosen export rate; source and backing gains explicit. |
| Output level | No hard clipping; lower gain with a visible report if needed. Loudness mastering is not automatic. |

Reject all-silent input. Warn, rather than claim certainty, about detected clipping, noise, reverb, or multiple voices. Such detection cannot certify that a recording is clean.

## 6. Voice enrollment and recording

Enrollment records include the target owner's permission and permitted use. Consent attestation is a workflow safeguard, not identity verification.

For the initial RVC model:

- Use lossless, dry recordings of one consenting person: the user.
- Cover intended singing registers and techniques rather than collecting duration alone.
- Keep original recordings and derived training slices separate.
- Split train/development/test by whole take, recording session, or song **before slicing** to prevent adjacent-slice leakage.
- Build the retrieval index from training features only.
- Select checkpoints using development conversion listening, not training loss alone; do not tune on the final held-out set.
- Retain model, index, encoder identity, sample rate, F0 setting, and training recipe as one versioned bundle.

A profile is marked **qualified** only for the languages, styles, and range represented in its evaluation. Importing a model successfully does not qualify its quality.

The current plan uses singing, not a speech-only dataset. For a 10-minute total budget, an initial proposed split is 8 minutes of training plus separate one-minute development and evaluation sets, divided by whole take/song before slicing. This leaves less training material than the low end of the published 10-30 minute recommendation; the pilot remains worthwhile but unqualified until evaluated. Five minutes can support an exploratory smaller pilot. Collect more only when a specific coverage/quality gap justifies it.

Include varied sung phrases, sustained vowels, comfortable low/mid/high registers, natural consonants, and breaths. Do not use duplicated clips to inflate the duration, reverb, heavy denoising, autotune, accompaniment, or uncomfortable forced notes. Preserve originals and inspect the actual training slices for lost breaths.

Speaking-only training remains technically possible, and speech may supplement phonetic coverage later. There is no assumed optimal singing/speech ratio. Do not change the current singing-based plan without a reason.

Target recordings do not need to contain the source song. For reference-only engines, follow their prompt recommendations: SoulX-Singer-SVC recommends clean singing; Seed-VC supports speech references. In both cases the source supplies pitch and timing.

## 7. Pitch preservation policy

The default and only P0 mode is `preserve_absolute_pitch`.

- Pitch shift is zero semitones.
- No octave matching based on source/target gender or estimated range.
- No automatic tuning to a musical scale.
- No duration scaling or tempo adjustment.
- No post-processing pitch shifter.
- Disable proposed-pitch/median-register adaptation and high-register octave-fold compatibility modes as well as visible semitone shifting.
- A range mismatch produces a warning and suggests better target data or another qualified configuration. It does not change the source notes.
- Pitch extraction errors and synthesis errors remain possible; the product states this explicitly.

If an optional transposition feature is added later, it must be a separately named mode with explicit user action and must not count toward unchanged-pitch acceptance.

### Breath and expression policy

Preserve the source performance's breath locations, phrasing, and vocal gestures. The target sample supplies identity, not a replacement performance timeline.

Breaths and many consonants have no stable pitch, so pitch accuracy cannot establish their quality. Do not aggressively denoise, gate, or replace them by default. Use the selected engine's unvoiced/protection behavior and assess audibility, timing, naturalness, and identity leakage. In the inspected Applio code, protection activates below `protect=0.5`; lower values restore more pre-retrieval source features on unvoiced frames. Tune on development clips and record the exact revision/value rather than assuming higher means more protection.

Exact reproduction of the target person's individual breathing or singing technique is not promised.

## 8. Nonfunctional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Native Windows 11 x64 is the first qualification target; exact GPU/driver/Python/Torch/engine versions are recorded. |
| NFR-02 | One GPU job at a time, including training and conversion. CPU/UI remain responsive. |
| NFR-03 | All runtime assets required by the selected workflow are predownloadable and hash-verified. Missing assets fail explicitly in offline mode. |
| NFR-04 | UI binds to loopback only; local session authorization and origin checks protect operations. No default network exposure. |
| NFR-05 | Completed artifacts are published atomically; partial outputs cannot be mistaken for completed work. |
| NFR-06 | Restart marks unfinished jobs interrupted. Completed output and history survive restart. |
| NFR-07 | Logs and diagnostics stay local, avoid raw audio contents and unnecessary personal identifiers, and are exported only by the user. |
| NFR-08 | Model code, weights, data, codecs, and bundled dependencies receive separate license/provenance review before distribution. |

## 9. Quality and performance acceptance plan

**All thresholds below are proposed engineering targets, not measured model capabilities or guarantees.** Freeze them before the final evaluation. If a target is missed, report it rather than changing the test or enabling transposition silently.

### Personal pilot versus broader qualification

The immediate handoff uses **one target voice (the user), four short held-out source clips, and one full song**, after module/GPU checks and training. It is a smoke/quality pilot, not product qualification. It does not require three target singers or eight raters. Use the [Windows handoff](WINDOWS_HANDOFF.md) to record personal acceptance, measurements, failures, and unknowns.

The multi-person protocol below is **optional later product qualification**, needed before claiming those broader quality gates were met. Personal preference or four successful conversions must not be reported as a blinded population study. Not running that broader study does not prevent the user adopting the existing app for personal use.

### Evaluation material for broader qualification

- Use three consenting target singers and at least two source singers.
- Collect 24 held-out source excerpts of 10-30 seconds: 12 typical dry-singing excerpts and 12 stress excerpts.
- Typical excerpts cover comfortable registers, sustained notes, consonants, silence, and ordinary vibrato.
- Include manually annotated breaths and unvoiced consonants in the typical group; inspect their starts/ends and audible texture separately from F0.
- Stress excerpts cover register extremes, fast runs, strong breathiness, range mismatch, and controlled reverb/noise or separation artifacts.
- Convert each excerpt to each target, yielding 72 cases per engine. Keep typical and stress groups separate in reports.
- References and target training data must not include the evaluation source recordings or held-out target identity references.
- Evaluate a minimum of three complete songs, including one near the 10-minute limit, for duration, memory, and chunk continuity.
- Never remove failed jobs from the denominator. Report exclusions with reasons decided before examining outputs.

### Proposed acceptance gates

| ID | Dimension | Initial P0 target |
|---|---|---|
| Q-01 | Absolute pitch | On each typical clip with enough voiced material: median absolute pitch error at most 25 cents on jointly voiced frames, and at least 90% of reference-voiced frames reproduced within 50 cents. Missing/unvoiced output frames count as failures for the latter. |
| Q-02 | Voicing | Voiced/unvoiced disagreement at most 5% of evaluated frames on typical clips. Tracker uncertainty is reported, not silently discarded. |
| Q-03 | Timing | Exported duration agrees within one sample at export rate; independently annotated vocal/note onsets have 95th-percentile absolute deviation at most 50 ms after only documented fixed pipeline-delay compensation. |
| Q-04 | Naturalness | Mean blinded human rating at least 3.5/5 on the typical group; no target subgroup mean below 3.0/5. Report distribution and uncertainty. |
| Q-05 | Target identity | At least 70% of typical-group judgments choose "probably same" or "definitely same" identity relative to held-out target references; no target subgroup below 60%. Include source-reference controls. |
| Q-06 | Lyrics | At least 95% of words judged preserved by listeners familiar with the language on typical clips. Source unintelligibility is separately annotated before conversion. |
| Q-07 | Artifacts | At least 90% of typical clips judged free of major audible artifacts by a majority of their raters. Major means an obvious dropout, click, unstable note, or severely mangled word. |
| Q-08 | Reliability | All 72 cases complete or produce explicit supported-input/quality errors; no crashes or silent fallbacks on typical cases. Full-song jobs preserve all source intervals. |
| Q-09 | Throughput | Target at most 3 minutes wall time for a 3-minute dry source, including model loading, inference, assembly, and export; installation, one-time downloads, and training excluded. |
| Q-10 | Preview | Target at most 30 seconds wall time for a 10-second preview under the same local job-start policy. Report cold-start separately; do not label process-per-job results as resident-model latency. |
| Q-11 | Memory | Complete the qualification corpus, including full songs, on the actual 12 GB card without OOM. Record GPU allocated/reserved peaks and total-device usage where available. |
| Q-12 | Offline | With network access blocked after setup, launch, preview, full conversion, guided training, export, and history work using the selected installed assets. |
| Q-13 | Breath/detail retention | At least 90% of annotated audible breath events in typical clips remain audible, with onset deviation at most 100 ms, and are judged free of obvious gating/metallic artifacts by a majority of raters. Report added breath events and consonant damage separately. |

No real-time guarantee is implied by Q-09. If only speed targets fail but quality passes, the product may remain useful for offline processing; any acceptance change requires an explicit scope decision.

### Measurement safeguards

Use an independent analysis pitch tracker, such as pYIN, rather than relying only on the engine's own F0 trace. Use manually reviewed synthetic tones and human-reviewed singing segments to identify tracker failures.

Do not octave-normalize, transpose, or time-warp outputs for acceptance. Pitch correlation and pitch-class/chroma similarity may be reported as diagnostics, not substitutes for absolute pitch error. Report voiced-frame coverage and dropouts alongside median errors.

For the broader blinded protocol, use at least eight listeners, including people familiar with the intended languages and some familiar with the targets. Assign balanced, randomized subsets so each evaluated output receives at least five ratings. Hide engine identity and settings. Play dry vocals before optional mixes so backing music does not conceal artifacts.

Report naturalness, identity, lyrics, and artifacts separately. Include real target recordings and unconverted sources as controls. Small-panel results are directional evidence, not population-wide quality guarantees.

## 10. Delivery phases and exit criteria

| Phase | Deliverable | Exit condition |
|---|---|---|
| A: Hardware spike | Pinned environment, diagnostics, one conversion, short local training run. | Real RTX 5070 execution confirmed; no CPU fallback; provenance/license inventory started. If no authorized inference model exists, do module/GPU checks first and finish conversion after training the user's model. |
| B: Quality experiment | Train/evaluate the selected RVC target voice; improve recording coverage or settings once if needed. | RVC meets agreed personal-use goals at unchanged pitch, or a documented failure triggers the contingency comparison. |
| C: Existing app or thin wrapper | Adopt the upstream app if sufficient; implement only missing P0 workflow features if justified. | Assess functional requirements and reliability/offline gates; disclose gaps and limited pilot evidence rather than claiming broad qualification. No custom model work. |
| D: Enhancements | Only approved P1 features. | Each feature is separately tested against the dry baseline and license policy. |

## 11. Decisions needed before implementation

Confirm available RAM/disk and Windows version, intended languages/styles, personal versus commercial distribution, and whether source vocals will normally be dry or already mixed with music. The initial recording commitment is already 5-10 minutes; additional recording is not a prerequisite to the pilot.

These decisions can change later priorities. They do not prevent starting the initial clean-vocal hardware and quality experiment. Do not expand into another engine, WSL2, or a custom UI without a documented need and explicit decision.
