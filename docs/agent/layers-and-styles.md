# Skill Layers and Style Playbooks

Reached from [`AGENT_GUIDE.md`](../../AGENT_GUIDE.md) when locating the skill behind a tool or technique, or choosing a visual style.

## Layer Map

Three instruction layers:

1. `tools/` — what exists, availability, cost, runtime, fallback, related skills.
2. `skills/` — how OpenMontage wants those tools used in pipelines.
3. `.agents/skills/` — raw vendor or technology knowledge.

Reading order: registry / tool contract → Layer 2 pipeline or creative skill (*what* and *when*) → Layer 3 vendor skill (*how*; mandatory before any generation call, per Rule Zero).

Example: before calling `kling_video`, read its `agent_skills` → `ai-video-gen` for Kling-specific prompt structure, camera direction syntax, and quality keywords.

For tool usage (prompts, parameters, usage patterns), read skills; they exist so the common case needs no implementation details. Read tool source for **debugging, audits, and verifying the governance contract**: when a skill and a tool disagree, or behavior differs from what the skill claims, the source is often the only way to catch a silent-availability bug or a stale docstring. After such a dive, fold the finding into the skill so the next agent skips it.

### Layer 3 skills, by category

When not arriving through a tool's `agent_skills` pointer, find the file by what you are trying to do:

| Category | Skills |
|---|---|
| **Composition runtime** | `remotion`, `remotion-best-practices`, `synthetic-screen-recording` (fake terminal/UI demos via Remotion TerminalScene) |
| **Animation knowledge (generic)** | `gsap-core`, `gsap-timeline`, `gsap-plugins` (SplitText / MorphSVG / DrawSVG / MotionPath / Flip / CustomEase), `gsap-utils`, `gsap-react`, `gsap-performance`, `gsap-scrolltrigger`, `gsap-frameworks`, `framer-motion` (Disney 12 principles), `lottie-bodymovin` (Lottie export) |
| **Character animation** | `character-rigging`, `svg-character-animation`, `pose-library-design`, `canvas-procedural-animation`, `character-animation-qa` |
| **Image generation** | `bfl-api`, `flux-best-practices` |
| **Video generation** | `seedance-2-0` (preferred premium default — cinematic, trailer, multi-shot, synced audio, lip-sync), `gemini-omni` (conversational video editing, reference tags, timecoded beats), `ai-video-gen`, `ltx2` |
| **Audio** | `elevenlabs`, `music`, `sound-effects`, `acestep`, `text-to-speech`, `setup-api-key` |
| **Speech-to-text** | `speech-to-text` (whisper `transcriber` — default, offline), `azure-speech-to-text` (optional cloud STT — tool `azure_stt`, preferred when `AZURE_SPEECH_KEY` is set) |
| **Avatar / lip-sync** | `avatar-video`, `heygen`, `create-video`, `faceswap`, `video-translate`, `agents` |
| **Capture** | `playwright-recording` (browser flows), `ffmpeg` (post) |
| **Visualization** | `beautiful-mermaid`, `d3-viz`, `manim-composer`, `manimce-best-practices`, `manimgl-best-practices` |
| **Media editing** | `video-edit`, `video-download`, `video-understand`, `video-toolkit`, `visual-style` |

Routing files, read first when in doubt:

- Animation runtime (Remotion primitives, GSAP plugins, framer-motion, Lottie, Manim, D3) → `skills/meta/animation-runtime-selector.md`.
- Screen-recording mode (real capture vs synthetic terminal) → `pipeline_defs/screen-demo.yaml` + `skills/pipelines/screen-demo/idea-director.md`.

## Style Playbooks

| Playbook | Best For |
|----------|----------|
| `clean-professional` | Corporate, educational, SaaS |
| `premium-minimalist` | Investor updates, expert explainers, product narratives |
| `flat-motion-graphics` | Social media, TikTok, startups |
| `minimalist-diagram` | Technical deep-dives, architecture |
| `ink-sketch` (Ink Theater) | Hand-drawn ink-on-white doodle animation; a character that draws itself, walks, dances; contraption explainers |

For custom, atelier, brand, launch, or hero work, read `skills/meta/taste-direction.md` before choosing a playbook and carry its `taste_profile` into the proposal, so later stages preserve the design read, visual variance, motion intensity, information density, reference strategy, and anti-patterns. Treat a playbook's `quality_rules` as constraints.

### Hand-drawn "doodle" animation → Ink Theater / Ink Puppet

For a hand-drawn ink doodle look ("a sketch that comes to life", "a stick figure that walks or dances", whiteboard-doodle explainers), use the **Ink Theater** engine + **Ink Puppet** mocap system (`skills/creative/ink-theater.md`, `ink-theater/README.md`). It is a style + reusable engine, not a pipeline: illustration / contraption pieces run on `animation`; a mocap character (draws itself → walks / dances / waves via `InkPuppet.choreograph([...])`) runs on `character-animation`. Entry points: **`/ink-art`** (create a vector doodle from scratch) and **`/animated-drawing`** (animate a *supplied* drawing with mocap, raster; `skills/creative/animated-drawing.md`). Character motion comes only from named mocap clips the agent chooses; hand-tuning it is out of scope.
