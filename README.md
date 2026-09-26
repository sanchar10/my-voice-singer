# My Voice Singer

Documentation and an experiment starter for **local singing voice conversion on a Windows RTX 5070 PC**. This repository contains research, proposed requirements, technical design, and a handoff for a separate Copilot CLI session on that PC. It is not an implemented application or a tested installer.

The goal is to replace the **original recording's singing identity with your own voice**, while retaining the original absolute pitch at the corresponding time, octave, melody, lyrics, phrasing, breath locations, and natural vocal detail as far as the model can achieve. Here, **source/original** means the performance being converted; **target voice** means your identity learned from your own recordings. Your training recordings do not need to contain the source song.

## Selected solution

Use **Applio with one target-specific, F0-enabled RVC model adapted from a compatible pretrained base**, ContentVec, RMVPE, the standard F0-enabled HiFi-GAN/NSF path, and a training-only retrieval index. Start with the existing local UI; do not build a custom application unless demonstrated workflow gaps justify it.

```text
Your clean singing recordings -> pretrained RVC adaptation + index -> personal voice model
                                                                          |
Original song -> dry source vocal -> zero-shift conversion -----------------+
                                         |
                                         +-> converted dry vocal
                                         +-> optional mix with aligned accompaniment
```

A finished song needs a usable solo vocal stem first; separation is a distinct, optional local step with its own artifacts and limitations. Keep the dry converted vocal for evaluation, and do not mix the original lead back in.

Aim for the upper end of the **5-10 minute** recording budget. Five minutes is exploratory. Applio recommends **10-30 minutes**, not a hard minimum or a guarantee; a proposed 8/1/1 split from 10 total minutes leaves only eight training minutes, before filtering. Split whole takes/songs before slicing and preserve natural breaths.

Lock pitch shift to **0**, autotune off, automatic/proposed target-range pitch adaptation off, high-register octave folding off, and no duration scaling or post-pitch effects. Exact pitch/time fidelity still requires independent measurement and dry listening.

**SoulX-Singer-SVC is only a contingency** after a bounded RVC data/settings iteration misses the goals, or if enrollment becomes reference-only. Archived Seed-VC is a legacy baseline, not the default.

## Reading order

1. [Windows handoff](docs/singing-voice-conversion/WINDOWS_HANDOFF.md): checkout instructions, copy/paste CLI kickoff, inputs, milestones, local result templates, and stop conditions.
2. [Product specification](docs/singing-voice-conversion/SPEC.md): confirmed goal, proposed scope, and explicitly unmeasured quality targets.
3. [Research and sources](docs/singing-voice-conversion/RESEARCH.md): selection rationale, source-level caveats, alternatives, licenses, and evidence limits.
4. [Technical design](docs/singing-voice-conversion/DESIGN.md): selected engine recipe and proposed architecture only if the existing workflow proves insufficient.

## What is not implemented or verified

No runtime code, installer, dependency lockfile, trained voice, downloaded weights, recordings, conversion output, or benchmark is included. This documentation handoff performed no audio-model execution or training on the Mac. Actual Windows/RTX 5070 compatibility, memory use, speed, perceptual quality, and offline behavior have **not** been measured. Numerical targets in the specification are proposals, not model capabilities.

The next session must inspect real hardware, resolve and pin a coherent current upstream environment, exercise an actual GPU operation and model path, then train and evaluate your voice locally. There is no automatic CPU/cloud fallback, no paid cloud processing, and no requirement to recruit a multi-person listening panel for the initial personal pilot.

## Repository and privacy boundaries

Keep recordings, weights, indexes, environments, local configuration, and experiment outputs **outside this checkout**. The [ignore rules](.gitignore) provide a backstop, not a privacy guarantee. Commit only documentation and deliberately reviewed, sanitized text manifests; never commit secrets or raw personal results. Use trusted, authorized model sources, and assess code, weights, dependencies, and audio rights separately. Do not upload your audio to public demos.
