# Local Singing Voice Conversion: Feasibility and Quality Research

Date: 2026-09-25, Pacific time

Status: Source-based research draft, adapted for repository handoff; no models or GPU benchmarks were run for this report.

Revision note: The user can provide 5-10 minutes of target singing. After a deeper source-level comparison, the selected default is Applio with a target-specific, F0-enabled RVC model. SoulX-Singer-SVC remains a reference-only contingency; Seed-VC is a legacy comparison.

Companion documents: [Windows handoff](WINDOWS_HANDOFF.md) | [Product specification](SPEC.md) | [Technical design](DESIGN.md) | [Repository overview](../../README.md)

**Terminology:** source/original means the singing performance being converted; target voice means the user's own identity learned from their recordings. Other singers appear in the literature and possible broader qualification, not as required participants in the personal pilot.

## 1. Executive recommendation

**Recommendation: use Applio with a target-specific, F0-enabled RVC model trained from a compatible pretrained base on the user's singing.** Start with the existing application, not a custom model or application. This is the strongest practical fit identified for the combined requirements: reusable target identity, unchanged pitch/timing, breath handling, Windows, and an RTX 5070.

This is an engineering selection based on inspected behavior, training workflow, and deployment risk. It is not a claim of independently measured perceptual superiority over every alternative.

The user's desktop RTX 5070 has 12 GB GDDR7 according to NVIDIA [R1]. This is a credible hardware platform for an RVC-based prototype, including per-voice training with appropriate batch sizes. Exact memory consumption, processing speed, and Windows compatibility must still be measured on this PC. It is not a guarantee that every singing model, training recipe, or unbounded audio input fits.

Recommended order:

1. Aim for the upper end of the user's 5-10 minute singing-recording budget and train one Applio/RVC target voice from pretrained weights.
2. Evaluate held-out dry singing for likeness, absolute pitch, timing, and breaths. Improve missing recording coverage or settings on development material before assuming a different engine is required.
3. Keep using the existing application if it meets the need. Build a thin wrapper only for missing workflow features after the personal quality and hardware gates in [WINDOWS_HANDOFF.md](WINDOWS_HANDOFF.md) pass; use [SPEC.md](SPEC.md) for optional broader product qualification.

Do not require an initial multi-engine implementation. Reopen the choice if Applio fails the user's quality goals after a bounded data/settings iteration, or if enrollment later becomes reference-only. In that contingency, SoulX-Singer-SVC is the newer candidate to compare.

No paid API, cloud GPU, or subscription is necessary for this approach. Initial downloads require internet access; conversion can subsequently be entirely offline if every dependency and model is cached and network-dependent features are disabled. Offline operation still needs verification.

**Expected outcome:** convincing creative/demo-quality conversions are plausible with suitable recordings and a well-matched target model. Consistently indistinguishable, studio-release-quality results across arbitrary singers, ranges, languages, and noisy recordings are not a defensible promise.

## 2. What the system changes

| Property | Intended behavior |
|---|---|
| Vocal identity/timbre | Change toward the target singer: the user. |
| Absolute pitch | Preserve the source notes, including their octave. |
| Melody and rhythm | Preserve note progression, timing, pauses, and phrasing. |
| Lyrics | Preserve the sung content; no transcription-and-regeneration stage. |
| Vibrato and expression | Preserve as far as the conversion model permits; assess separately. |
| Breaths and unvoiced consonants | Preserve their occurrence and timing while seeking natural converted texture; exact target-specific breath identity is not guaranteed. |
| Target singer's personal interpretation | Not promised: this is not a simulation of how that person would independently perform the song. |
| Source singing mistakes | Remain unless the user later opts into a separate editing feature. |

The target does not need to sing the same song as the source. However, target recordings that cover similar registers, phonemes, languages, and techniques are a better basis for evaluation than an unrelated, narrow speaking sample.

"Same tune" must not mean "same pitch class after an octave shift." Many cover workflows deliberately transpose voices into another register. That violates the user's requirement here.

## 3. Candidate comparison

These are source-supported capabilities, not a ranking established on the user's recordings.

| Candidate | Target enrollment | Relevant evidence | Limitations and recommendation |
|---|---|---|---|
| **Applio / RVC** | Train or import a target-specific model; retrieval index recommended. | Windows UI and CLI; explicit pitch extraction; training guide recommends 10-30 minutes of clean audio [R3-R8]. | Selected default now that the user can record singing. Start near 10 minutes and measure quality. Applio says future updates will mainly be maintenance. |
| **SoulX-Singer-SVC** | Zero-shot target reference; its UI recommends clean singing. | Official SVC release dated 2026-03-16; waveform/F0 conversion without lyric/MIDI transcription; local UI and released checkpoint [R19-R21]. | Reference-only contingency, not the current primary recommendation. UI enables automatic pitch shift by default. Dependencies, 12 GB memory fit, long-song stability, and exact pitch/breath behavior require testing. |
| **Seed-VC v1 singing checkpoint** | Zero-shot reference; optional fine-tuning. README advertises references of 1-30 seconds. | Dedicated 44.1 kHz, F0-conditioned singing model; singing evaluation against RVC; Windows instructions [R9-R11]. | Legacy comparison, not the best-current recommendation. Repository is archived; dependency file is problematic for modern GPUs. Do not substitute its newer speech/style model for the singing checkpoint. |
| **Vevo2, style-preserved FM path** | Reference-conditioned conversion. | Explicitly supports style-preserved SVC; released checkpoints and inference recipe [R15]. | Research alternative, not dismissed as speech-only. Exact pitch/timing fidelity and 12 GB Windows performance remain unverified. Checkpoint license is noncommercial/no-derivatives [R16]. Not the default product dependency. |
| **FreeSVC** | Zero-shot speaker representation. | Official multilingual SVC code, F0 conditioning, and pretrained checkpoints [R22]. | Additional research baseline, not a newer-than-Seed successor established here. Released weights are noncommercial; README warns that the preliminary checkpoint was mainly fine-tuned on speech. |
| **Original so-vits-svc 4.1** | Target-specific training. | Singing-focused architecture and pitch conditioning [R14]. | Original repo is archived, has AGPL licensing and additional usage language. No compelling reason to choose it first for a new Windows product. Forks require separate review. |

R2-SVC is also a relevant research direction: its authors address pitch errors and artifacts from separated music [R18]. The reviewed paper/demo does not establish a qualified Windows RTX 5070 distribution, checkpoint license, or measured 12 GB deployment. It remains a watchlist item, not an implementation dependency.

### Why start with Applio/RVC?

- The project provides an existing local workflow, so the first experiment does not require custom UI development.
- Explicit F0 conditioning and a zero-semitone setting match the core requirement.
- Target-specific training is useful when the same small set of voices will be reused.
- Training data, index strength, consonant protection, and pitch extraction can be adjusted and compared.
- Its code licensing is more straightforward than the other shortlisted alternatives, subject to the complete dependency/weight review below.

This is a recommendation about implementation risk and workflow fit, not proof that RVC sounds better than every zero-shot model.

### Deeper comparison behind the final selection

| User priority | Inspected Applio/RVC behavior | Inspected SoulX-Singer-SVC behavior | Decision implication |
|---|---|---|---|
| Use the available 5-10 minutes | Documented per-target training plus retrieval-index construction uses the prepared dataset [R5, R28]. | Released UI uses a prompt cropped to at most 30 seconds [R21]. Reviewed official workflow is inference-focused, not an equivalent documented per-target training workflow. | RVC directly exploits the user's willingness to record a reusable training set. This does not prove every trained voice beats every zero-shot result. |
| Absolute pitch control | Pipeline retains a continuous F0 contour alongside coarse pitch features; standard F0-enabled HiFi-GAN path feeds the continuous contour into the NSF excitation [R24-R25]. | SVC conditions on discretized F0 embeddings; inspected code uses 20-cent bins and a 20 ms analysis grid [R26]. | RVC offers a direct pitch-control path suited to the requirement. Neither architecture guarantees exact output F0. |
| Preserve breath events | Unvoiced protection blends pre-retrieval source features back on unvoiced frames [R24]. | Long-track segmentation filters intervals with very little detected voicing, leaving omitted intervals zero in the assembled track [R26]. | RVC provides an explicit relevant control. SoulX needs extra checks for isolated breath/unvoiced spans; this does not imply ordinary breaths within singing segments always disappear. |
| Keep the timeline | Native pipeline extracts the padded track's F0 and slices aligned feature/audio regions [R24]. | Generated segments are placed on a source-length canvas, and individual outputs are cropped/padded to input length [R26]. | Both need onset and boundary tests. Identical file length alone cannot demonstrate preserved articulation. |
| Local repeatable workflow | Existing target training, inference, index, and Windows workflow [R3-R8]. | Existing inference UI, but older Torch pins and unqualified full-pipeline VRAM [R21]. | Prefer RVC for the first personally trained local voice. |

Additional RVC caveats found during inspection:

- The protection mechanism preserves source features, not a perfect recording of the target person's breaths. Increased naturalness can trade off against complete identity conversion. In the inspected code, it activates below `protect=0.5`, with lower values restoring more pre-retrieval source features on unvoiced frames; "higher means more protection" is not a safe interpretation.
- The inspected code uses a 10 ms analysis hop; that is a conditioning grid, not an audio-accuracy guarantee.
- Pitch-estimator octave errors, high-register limits, and source noise still matter. The current predictor code includes an optional high-register compatibility/folding mode [R27]; keep that mode disabled for unchanged absolute pitch.
- Training preprocessing can slice/trim quiet regions. Inspect the actual retained slices so meaningful breaths are not unintentionally removed.
- The source can still have synthesis artifacts, boundary drift, or clipped details. Local qualification remains necessary.

Newer model size, a finer/coarser representation, or a higher output sample rate alone is not sufficient to predict perceptual quality. The selection uses these observations together with workflow and data fit.

### Existing end-to-end workflows: no new model required

| Tool | Ready-made local workflow | What still needs work |
|---|---|---|
| Applio | Prepare/train a target model using its training workflow, select model and source audio, convert, audition, and export through a browser UI. | Initial setup, authorized target recordings/model, settings selection, and quality qualification. It does not clone a new target from an arbitrary short reference alone. |
| SoulX-Singer-SVC | `webui_svc.py` accepts a voice prompt plus singing audio, with optional separation and accompaniment mixing. No target-specific training or lyric/MIDI transcription is required for SVC. | Disable automatic pitch shift and keep manual shift at zero. Validate installation and full pipeline on the RTX 5070; do not assume its SVS benchmark establishes SVC quality. |
| Seed-VC | Dedicated singing UI, `app_svc.py`: source singing plus target reference, then conversion and output. No target-specific training is required for its zero-shot path. | Resolve the archived project's dependency issues, select the actual singing checkpoint, and qualify pitch/breath quality. |
| Vevo2 | Released style-preserved singing-conversion inference pipeline. | More research-oriented integration; Windows/12 GB qualification and checkpoint-use restrictions remain. |

"End to end" here means a vocal recording plus an enrolled/reference target can produce converted vocal audio without implementing encoders, pitch extraction, synthesis, or training algorithms ourselves. It does not imply perfect fidelity, a tested one-click RTX 5070 installer, or automatic removal of every instrument and backing singer from a finished mix.

### Why the recommendation changed

The initial research missed SoulX-Singer-SVC, a released 2026 candidate that fits the source-audio-plus-reference workflow. That omission made the earlier Seed-VC-first advice too narrow. The SoulX official repository is not archived, and its news section dates the SVC release to March 16, 2026 [R19]. Its model repository actually contains `model-svc.pt`; this is not merely a paper or promised release [R20].

This does not justify replacing one untested "best" claim with another. The reviewed SoulX SVC UI recommends a clean singing prompt, warns of unstable identity on long/wide-range songs, and enables automatic pitch shifting by default [R21]. Its requirements pin Torch 2.2.0, which is not the Blackwell-ready stack described in this report. Qualify a compatible environment rather than assuming the newer release is plug-and-play.

The main SVC checkpoint is about 2.79 GB on disk [R20]. Disk size is not peak VRAM: activations, preprocessing models, vocoder, precision, and duration all contribute. No 12 GB full-pipeline measurement was established here.

### The user's three challenges

**Target timbre:** infer a target representation from a reference or learn it through target-specific training. The acoustic content of the source and target identity cannot be perfectly disentangled, so identity leakage and register-dependent similarity must be assessed.

**The right pitch at the right time:** preserve the source's absolute F0 trajectory and alignment, not only the melody shape or output file duration. Use zero transposition, disable automatic target-range matching, and check instantaneous pitch plus note/lyric timing. Explicit conditioning helps; it does not mathematically guarantee identical synthesized pitch.

**Breathing and vocal detail:** breaths, aspiration, and unvoiced consonants are not fully represented by an F0 curve. Most breaths have no stable fundamental frequency. Existing engines synthesize these sounds as part of the audio pipeline; Applio also exposes unvoiced/consonant protection [R6]. Preserve event timing and judge naturalness separately from voiced pitch. Protection may retain some source character, while aggressive conversion or cleanup may damage the sound. No reviewed tool establishes perfect transfer of every breath, rasp, vibrato, and register transition.

The intended result is the source performance in the target's vocal identity, not moving breaths to where the target reference happened to breathe. Do not paste the reference's breath waveform into the song or hard-gate source breaths as a default repair.

## 4. What the published quality evidence actually shows

### Seed-VC's singing-specific comparison

Seed-VC's `EVAL.md` reports a comparison using M4Singer source material, four target voices, and trained RVCv2-F0-48k baselines [R10].

| Reported aggregate metric | RVCv2 | Seed-VC | Interpretation |
|---|---:|---:|---|
| F0 correlation, higher preferred | 0.9404 | 0.9375 | Both track pitch variation strongly in this experiment. |
| Speaker embedding cosine similarity, higher preferred | 0.7264 | 0.7405 | Small advantage for Seed-VC under this embedding metric. |
| Character error rate, lower preferred | 28.46 | 19.70 | Seed-VC scores better under the experiment's singing ASR evaluation. |
| DNSMOS overall prediction, higher preferred | 3.12 | 3.06 | Slight advantage for RVC under this automated quality predictor. |

Important qualifications:

- These are the authors' results, not independently reproduced measurements here.
- The setup uses only four target voices. RVC training quality and reference selection affect the comparison.
- The authors apply +12 or -12 semitone shifts to some cross-gender conversions. Consequently, the aggregate table does **not** validate unchanged absolute pitch for this product.
- F0 correlation is not "94% pitch accuracy." A strongly correlated contour can still have a systematic pitch offset.
- An embedding score of 0.74 is not "74% indistinguishable."
- DNSMOS is an automated predictor, not a human listening MOS. Speech-oriented predictors and ASR have limitations on singing.
- The benchmark's third-party voice datasets are evidence sources, not recommended enrollment data. Use consenting participants for this project.

### Broader singing research

The extended SVCC 2025 analysis evaluates singing identity and singing-style conversion [R17]. It reports that matching singing style and naturalness remains difficult and explains why objective scores do not replace human listening. Some systems approach the reference in particular identity comparisons, but this is not proof of universal indistinguishability.

Its task differs from this product's timbre-only, unchanged-performance goal. Challenge results therefore support the feasibility of the field and the need for careful evaluation, not a numerical quality forecast for Applio on an RTX 5070.

### Evidence still missing

- A controlled, same-data, zero-transposition comparison of the shortlisted engines on the user's singers.
- End-to-end latency and peak memory measurements on this exact PC.
- Quality across the user's intended languages and musical styles.
- Whether short speaking references are sufficient for the particular target identities.
- Whether output remains convincing as an exposed vocal rather than only inside a backing mix.

## 5. Realistic quality expectations

The following are engineering expectations, not measured product scores.

| Recording/conversion condition | Reasonable expectation | Likely failure modes |
|---|---|---|
| Dry, single singer; moderate register; well-recorded target training data | Strongest chance of recognizable, pleasant conversion suitable for demos and creative use. | Occasional consonant roughness, metallic vowels, breaths, or identity leakage. |
| Clean source; short, clean target reference only | Fast way to assess target resemblance; potentially convincing but more variable. | Source voice leaking through, inconsistent identity, reference sensitivity. |
| Target reference is speaking only; source is expressive singing | Useful experiment, not a reliable promise of how the target sings. | Weak high-register identity, unnatural register transitions. |
| Source and target differ substantially in range or technique | Possible, but retaining exact notes may conflict with a natural target timbre. | Falsetto/belt artifacts, unstable harmonics, exaggerated formants. |
| Vocals separated from a finished mix | Often usable for rough covers; below the dry-input ceiling. | Reverb, instrument bleed, missing consonants, original singer remaining in accompaniment. |
| Harmonies, choir, doubled lead, screams, growls, extreme melisma | Outside the initial supported scope. | Incorrect pitch tracking, voice blending, rhythm changes, unstable synthesis. |

High sampling rate is not a quality guarantee. Writing a 48 kHz file does not make a 44.1 kHz model output more detailed, and it does not remove model artifacts.

The GPU affects speed and feasible model/settings choices; it does not fix missing target data, poor pitch extraction, or inadequate model generalization.

## 6. Target recording guidance

The user is willing to record **5-10 minutes of singing**. Start there, preferably near 10 minutes; do not demand 30 minutes before a first experiment. Applio's published recommendation remains **10-30 minutes** [R5], so a shorter training set is an exploratory pilot, not a documented quality guarantee.

| Available material | Practical treatment |
|---|---|
| 5 minutes total | Enough to attempt a pretrained-model pilot; expect less register/phoneme coverage and reserve some material rather than training on every take. |
| 10 minutes total | Preferred first recording budget. A proposed split is 8 minutes training, 1 minute development, 1 minute held-out evaluation. Eight training minutes is below the guide's recommended range; label it honestly. |
| More material later | Collect targeted missing registers, consonants, or vocal techniques if the pilot exposes gaps. Do not add duration solely by duplicating clips. |

Suggested 10-minute capture plan, not a standardized requirement: roughly five minutes of varied sung phrases, two minutes of sustained vowels/simple comfortable scales, one minute of additional comfortable register/technique coverage, and two minutes of separate development/evaluation takes. Preserve natural breaths. Divide by whole takes before slicing; distribute registers across the splits.

Prefer 10 minutes of clean, varied singing to a longer noisy or highly repetitive recording. Count usable phrase material after cleanup, not simply wall-clock recording time. No need to sing the source song, and no need to force uncomfortable notes.

Fine-tune/adapt compatible pretrained weights rather than training a foundational voice model from random initialization. More data can help, but it does not automatically fix F0 extraction errors, poor settings, or unsupported vocal techniques.

### Speaking versus singing

**Speaking recordings can be used; singing is not an absolute prerequisite for every SVC system.** The original RVC documentation explicitly recommends low-noise speech as usable training material [R3], and SVCC 2023 included a cross-domain task with speech-only target training data and singing source input [R23].

For this product, prioritize clean target singing because it demonstrates sustained vowels, register transitions, aspiration, and techniques absent or underrepresented in ordinary speech. Optional clean speech can supplement phonetic coverage, but there is no established universal singing/speech ratio in this report. Whether mixing helps a particular engine is an experiment, not an assumption.

The source singing provides the melody and timing. A spoken target sample does not need to contain that melody. What speech-only enrollment leaves uncertain is the target's singing timbre and behavior across registers, not the source song's notes.

For reference-only inference, follow the chosen model: SoulX's SVC UI explicitly recommends a clean singing prompt [R21]; Seed-VC advertises speech references [R9]. These are inference references, not a requirement to train a new model. Neither target training nor reference audio needs to be the same song as the source.

- Record one person at a time with no accompaniment, doubled vocals, obvious room echo, clipping, reverb, autotune, or heavy processing.
- Include comfortable low/mid/high registers, sustained vowels, lyrics, consonants, and some examples of the intended techniques.
- Keep the microphone and room reasonably consistent. Do not aggressively denoise away vocal detail.
- Preserve original lossless recordings. Derive training slices separately and inspect them for lost breaths.
- Hold out entire takes, recordings, or songs before slicing; do not randomly split neighboring slices from the same take across training and testing. Build the retrieval index from training only.
- Reserve unfamiliar material for final listening so memorization does not masquerade as generalization.

If the optional Seed-VC legacy comparison is ever chosen, begin with several different clean 10-30 second target references and compare them on development material. The README's 1-second lower bound is a capability claim, not our recommended quality target. Short-reference inference avoids target-specific training; it does not eliminate the need for good reference material.

## 7. Windows and RTX 5070 feasibility

There is stronger support than the blanket advice "Blackwell needs custom or nightly PyTorch."

- PyTorch introduced Blackwell support with its 2.7/CUDA 12.8 release [R2].
- The inspected RVC README explicitly documents an RTX 50-series path using Python 3.12 and `torch/torchaudio 2.7.1+cu128` [R3].
- The inspected Applio Windows installer creates a Python 3.12 environment and uses a CUDA 12.8 package index; its current requirements pin a newer Torch pair [R8].
- Seed-VC's requirements combine old Torch 2.4 pins with nightly CUDA 12.6 declarations [R11]. That file is not a validated RTX 5070 installation recipe.
- SoulX-Singer's requirements pin Torch/audio 2.2.0 and additional specialized dependencies [R21]. Its 2026 release date does not make that dependency set automatically compatible with an RTX 5070; the complete SVC/preprocessing stack needs qualification.

These facts establish plausible installation routes, not that an arbitrary downloaded Windows bundle works. Use a coherent, pinned engine/environment pair. Do not blindly upgrade one dependency in an old environment, mix requirements from different projects, or install into global Python.

The complete CUDA Toolkit is not automatically necessary when using compatible prebuilt PyTorch wheels; a compatible NVIDIA driver is still required. Additional compilation dependencies may be needed if a selected package builds custom GPU extensions. Driver-reported CUDA capability is not the installed PyTorch runtime, and CUDA availability alone does not demonstrate working kernels or model execution.

Planning assumptions, to verify:

| Resource | Proposed planning provision |
|---|---|
| GPU | Desktop RTX 5070, 12 GB; one GPU job at a time. |
| System RAM | Prefer 32 GB; 16 GB may suffice for a smaller workflow but is not yet qualified. |
| Free SSD | Initially reserve 50 GB for environments, model weights, datasets, features, and checkpoints; measure actual usage. |
| OS | Target Windows 11 x64 first; exact user version remains unknown. |
| Runtime | Native Windows first. Consider WSL2 only if a selected engine has a demonstrated native-Windows blocker and the user explicitly chooses it. |

Do not promise a training duration or seconds-per-song before measuring. Per-voice RVC training is a reasonable local objective; training a foundation voice model from scratch is not part of this proposal.

## 8. Licensing, privacy, and distribution

| Component | Verified source-level finding | Required action |
|---|---|---|
| Applio | MIT license; README also points to Terms of Use and states repository code/weights are MIT [R4]. | Preserve notices; review terms, downloaded base models, encoders, and all other dependencies independently. |
| Original RVC | MIT indicated in repository documentation [R3]. | Verify the exact pinned revision and each distributed artifact. |
| SoulX-Singer-SVC | Official repository states Apache 2.0 for code and weights; model card also declares Apache 2.0 [R19-R20]. | Verify every preprocessing/separation/vocoder dependency separately. The main-model license is not blanket clearance for all downloaded components. |
| Seed-VC | Code license is GPL v3; official Hugging Face model card declares `gpl-3.0` [R13]. | Evaluate copyleft obligations before integration/distribution. GPL is not a noncommercial license. |
| Original so-vits-svc | AGPL v3 plus additional README usage language [R14]. | Do not treat it as MIT or presume unrestricted product use. |
| Vevo2 | Amphion code is MIT; model card declares `cc-by-nc-nd-4.0` [R15-R16]. | Do not assume code license grants commercial or derivative rights to weights. Seek clarification/permission before such use. |
| FreeSVC | Code is MIT; released weights are CC BY-NC-SA 4.0 according to its README [R22]. | Keep noncommercial-weight restrictions separate from the code license. |

A paper's publication license is not the license of its implementation or weights. Running a model in another process does not automatically avoid license obligations. Review the intended distribution, not just the architecture.

Record target-owner consent and permitted uses. Treat recordings, learned voice models, indexes, and reference embeddings as sensitive data. No cloud calls, model marketplace uploads, telemetry, or public sharing links are required. Audio rights and permission to imitate a voice are separate considerations. These are engineering safeguards, not legal advice.

## 9. Recommended decision gates

1. **Hardware gate:** Actual GPU tensor operation, 30-second conversion, full-song conversion, and short training smoke run succeed without CPU fallback or out-of-memory errors. If no authorized inference model is available, do module/GPU checks and then train the user's model before conversion.
2. **Quality gate:** Independent absolute-pitch/timing checks and dry listening on held-out clean singing; stress cases reported separately. Personal acceptance is distinct from the optional broader blinded protocol.
3. **Product gate:** The selected engine's data needs, installation steps, licenses, and observed speed fit the user's intended workflow. Build no custom app until a real gap warrants it.

The detailed thresholds are proposed in [SPEC.md](SPEC.md). They are not claims that the models already meet them. The [Windows handoff](WINDOWS_HANDOFF.md) defines the smaller one-user pilot without requiring a multi-singer panel.

Budget roughly 2-5 engineering days for setup and a first baseline, then 1-2 weeks for recording, controlled comparisons, and iteration. A thin usable local application, only if later justified, may take a further 2-4 weeks for one experienced developer. These are planning estimates, not commitments or measured performance; dataset collection and Windows dependency issues can dominate.

## 10. Sources and audit notes

The originating research inspected these sources on 2026-09-25 Pacific time / 2026-09-26 UTC. This handoff preserves that source audit; it is not a new hardware or model evaluation. Repository branches and documentation are mutable. Freeze commit IDs, artifact hashes, and license copies during the Windows setup.

- **R1:** [NVIDIA RTX 5070 family specifications](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/rtx-5070-family/) - desktop memory and architecture table.
- **R2:** [PyTorch 2.7 release](https://pytorch.org/blog/pytorch-2-7/) - introduction of Blackwell/CUDA 12.8 support; not a complete application compatibility certificate.
- **R3:** [RVC English README](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI/blob/main/docs/en/README.en.md) - training guidance, Windows and RTX 50-series installation.
- **R4:** [Applio README](https://github.com/IAHispano/Applio), [LICENSE](https://github.com/IAHispano/Applio/blob/main/LICENSE), [Terms of Use](https://github.com/IAHispano/Applio/blob/main/TERMS_OF_USE.md), [repository metadata](https://api.github.com/repos/IAHispano/Applio) - maintenance and licensing.
- **R5:** [Applio training guide](https://docs.applio.org/getting-started/training/) - 10-30 minute dataset recommendation, training, retrieval index.
- **R6:** [Applio inference guide](https://docs.applio.org/getting-started/inference/) - available controls. Generic pitch-shift/autotune recommendations are deliberately not adopted for this product.
- **R7:** [Applio architecture](https://docs.applio.org/reference/architecture/) - F0, embeddings, retrieval, generator, and model compatibility.
- **R8:** [Applio Windows installer](https://github.com/IAHispano/Applio/blob/main/run-install.bat), [requirements](https://github.com/IAHispano/Applio/blob/main/requirements.txt), [installation guide](https://docs.applio.org/getting-started/installation/) - environment details; website and current script differ on Python version.
- **R9:** [Seed-VC README](https://github.com/Plachtaa/seed-vc/blob/main/README.md) - singing checkpoint, reference lengths, controls, and fine-tuning.
- **R10:** [Seed-VC EVAL.md](https://github.com/Plachtaa/seed-vc/blob/main/EVAL.md) - exact reported singing metrics and transposition caveat.
- **R11:** [Seed-VC requirements](https://github.com/Plachtaa/seed-vc/blob/main/requirements.txt) - conflicting/old Torch declarations.
- **R12:** [Seed-VC repository metadata](https://api.github.com/repos/Plachtaa/seed-vc) - `archived: true` when inspected.
- **R13:** [Seed-VC LICENSE](https://github.com/Plachtaa/seed-vc/blob/main/LICENSE), [official model card](https://huggingface.co/Plachta/Seed-VC) - GPL v3 declarations.
- **R14:** [so-vits-svc README](https://github.com/svc-develop-team/so-vits-svc), [LICENSE](https://github.com/svc-develop-team/so-vits-svc/blob/4.1-Stable/LICENSE), [repository metadata](https://api.github.com/repos/svc-develop-team/so-vits-svc) - singing model, AGPL, archive status.
- **R15:** [Vevo2 official implementation](https://github.com/open-mmlab/Amphion/tree/main/models/svc/vevo2), [Amphion LICENSE](https://github.com/open-mmlab/Amphion/blob/main/LICENSE) - supported tasks and model components.
- **R16:** [Vevo2 official model card](https://huggingface.co/RMSnow/Vevo2) - checkpoint license and model descriptions.
- **R17:** [Extended SVCC 2025 analysis](https://arxiv.org/html/2509.15629v2) - task definitions, human evaluation, and limitations of objective measures.
- **R18:** [R2-SVC author page](https://c9412600.github.io/svc/), [paper](https://arxiv.org/html/2510.20677v1) - robustness research, not a qualified product release.
- **R19:** [SoulX-Singer official README](https://github.com/Soul-AILab/SoulX-Singer), [repository metadata](https://api.github.com/repos/Soul-AILab/SoulX-Singer) - SVC release date, capabilities, stated licensing, and non-archived status.
- **R20:** [SoulX-Singer model card](https://huggingface.co/Soul-AILab/SoulX-Singer), [actual SVC checkpoint](https://huggingface.co/Soul-AILab/SoulX-Singer/blob/main/model-svc.pt) - downloadable weights and declared Apache 2.0 license. Model-card prose mainly describes SVS; use the code README for SVC-specific capabilities.
- **R21:** [SoulX SVC UI source](https://github.com/Soul-AILab/SoulX-Singer/blob/main/webui_svc.py), [requirements](https://github.com/Soul-AILab/SoulX-Singer/blob/main/requirements.txt), [preprocessing guide](https://github.com/Soul-AILab/SoulX-Singer/blob/main/preprocess/README.md) - singing prompt advice, pitch-shift defaults, duration handling, environment requirements.
- **R22:** [FreeSVC official repository](https://github.com/freds0/free-svc) - downloadable model links, preliminary speech-heavy checkpoint warning, and weight restrictions.
- **R23:** [SVCC 2023 official task definitions](https://vc-challenge.org/svcc2023/index.html) - separate target-singing and target-speech training tasks. Demonstrates speech-only SVC as a valid setting, not equal quality for every model.
- **R24:** [Applio inference pipeline](https://github.com/IAHispano/Applio/blob/main/rvc/infer/pipeline.py) - continuous/coarse F0, aligned segmentation, retrieval, and unvoiced protection implementation.
- **R25:** [Applio synthesizer](https://github.com/IAHispano/Applio/blob/main/rvc/lib/algorithm/synthesizers.py), [NSF generator](https://github.com/IAHispano/Applio/blob/main/rvc/lib/algorithm/generators/hifigan_nsf.py) - actual F0-enabled decoder selection and continuous F0 excitation.
- **R26:** [SoulX-Singer-SVC model implementation](https://github.com/Soul-AILab/SoulX-Singer/blob/main/soulxsinger/models/soulxsinger_svc.py), [configuration](https://github.com/Soul-AILab/SoulX-Singer/blob/main/soulxsinger/config/soulxsinger.yaml) - F0 quantization, long-track voiced-segment filtering, and output placement/padding. These are code observations, not measured perceptual comparisons.
- **R27:** [Applio F0 predictors](https://github.com/IAHispano/Applio/blob/main/rvc/lib/predictors/f0.py), [training preprocessing](https://github.com/IAHispano/Applio/blob/main/rvc/train/preprocess/preprocess.py) - high-register compatibility controls and dataset slicing.
- **R28:** [Applio pretrained-model guide](https://docs.applio.org/getting-started/pretrained/) - starting small-data target training from a pretrained generator/discriminator pair.

Selected inspected file fingerprints, recorded as Git blob IDs rather than commit IDs:

| File | Git blob ID |
|---|---|
| Applio `run-install.bat` | `53ec5151eec5c8fd80b8d1e6558efb901c7a69d2` |
| Applio `requirements.txt` | `b305accd3df7bc6b9dd277a8343bf6502b2c2f8f` |
| Seed-VC `README.md` | `2caf62fdedd40ee83aecae0aad80ca9933b222e0` |
| Seed-VC `EVAL.md` | `c93a49170ce48bcf8a61328f1f14b749f7ce2b90` |
| Seed-VC `requirements.txt` | `399c6649b58b757c3096351ef79a0cfa218390e8` |
| Vevo2 `README.md` | `5cf73a38418a66e94c2c606ceeb28fd4925d44ca` |
| SoulX-Singer `README.md` | `7f333fa7940aeed81a9bd18e7ffeee79d749f671` |
| SoulX-Singer `webui_svc.py` | `52fe0c4127341064626ac38f1d62bd79b847ba1b` |
| SoulX-Singer `requirements.txt` | `ca14955f697624d723c6e0d3bf163f0c53c16706` |
| Applio `rvc/infer/pipeline.py` | `e98a101e242ed04326cebca78a56aeb030265c73` |
| Applio `rvc/lib/algorithm/synthesizers.py` | `e920b535e9649106283dc3ad48d595b9ba6f9531` |
| SoulX-Singer `soulxsinger/models/soulxsinger_svc.py` | `1cfbe62c31c94dc7c51819c847295e176251065e` |

These are historical file-identity observations, **not engine commit pins, SHA-256 model hashes, or a dependency lockfile**. None authorize executing arbitrary checkpoint files.

Search summaries were used for discovery only. Conflicting claims about licensing, Windows support, and model capabilities were resolved against the primary files above in the originating research. No installation instructions from this report should be treated as a tested lockfile.
