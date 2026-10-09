# Composition and Visual Direction

Reached from [`AGENTS.md`](../../AGENTS.md) when designing the visual approach at proposal, and whenever a stage touches `video_compose`. The runtime-selection rule and runtime lock live in `AGENTS.md` → "Present Both Composition Runtimes (HARD RULE)".

## Composition Runtimes (Inside video_compose)

`video_compose` has three parallel, unranked engines. The choice is locked at proposal (`proposal_packet.production_plan.render_runtime`) and carried unchanged in `edit_decisions.render_runtime`, which `video_compose` dispatches on. Availability and per-engine notes: `registry._tools['video_compose'].get_info()` (`render_engines`, `remotion_note`, `hyperframes_note`).

- **FFmpeg**: video-only cuts, concat, trim, subtitle burn.
- **Remotion**: React scenes: stills → animated video, text/stat cards, charts, callouts, spring transitions, word-level captions, TalkingHead avatar.
- **HyperFrames**: HTML/CSS/GSAP: kinetic typography, product promos, launch reels, website-to-video, registry blocks, SVG character rigs.

Pick by concept: HyperFrames for motion-graphics-heavy ideas HTML/GSAP expresses naturally, Remotion for briefs that reuse the React scene stack. `skills/core/hyperframes.md` holds the decision matrix.

When Remotion is available, plan around it: Remotion animated scenes for `flat-motion-graphics` explainers, stat cards and charts for data-driven videos, spring animation for still images rather than FFmpeg pan-and-zoom. For a CLI/terminal/install demo use `TerminalScene` (synthetic screen recording, `.agents/skills/synthetic-screen-recording/SKILL.md`): deterministic and privacy-safe; reserve real capture (`screen_recorder`, `cap_recorder`, `playwright-recording`) for a real app UI or unpredictable live behavior. When Remotion is unavailable and `render_runtime="remotion"` was not locked, `video_compose` may fall back to FFmpeg Ken Burns on stills: name that tradeoff in the proposal.

Stock scene types (`cut.type`) and their schemas: `remotion-composer/SCENE_TYPES.md`. They are the **templated** path.

## Composition Authoring Mode — Templated vs Atelier

Orthogonal to *runtime* is *authoring mode*. Present it as its own proposal decision and log it in `decision_log` (`category: "composition_mode"`).

- **Templated**: assemble stock `cut.type` scenes into the `Explainer`/`CinematicRenderer` compositions. Fast, cheap, reliable, and the reason most videos look alike. Right for batch output, localization variants, quick drafts, low-stakes internal clips.
- **Atelier**: hand-author the composition: bespoke scenes, a one-off theme, motion written for this piece, rendered via `composition_mode: "atelier"` (`video_compose` → `_render_via_atelier`).

**Default to atelier for hero work**: marketing, launches, brand pieces, any single-deliverable explainer that must impress. The deciding rule: *reuse engine knowledge, never creative components.* In atelier mode the stock scene catalog, `hyperframes-registry` blocks, fixtures, and finished components are a *mechanics codex* to read, never parts to assemble: they are frozen looks that reintroduce sameness.

Atelier sequence:

1. `skills/meta/taste-direction.md`: set the design read and taste dials.
2. `skills/meta/bespoke-composition.md`: art direction (`visual-style`) → motion principles (Disney 12 via `framer-motion`/`lottie-bodymovin`) → engine mechanics (`remotion-best-practices`, stock components as codex) → render via the atelier path.
3. Close with a **distinctness review**: *could this be any other product's video? does it reuse a look I've made before?*

Atelier costs more tokens and iteration than templated; say so at proposal so the user opts in knowingly.

## Style Playbooks

List them: `grep -H "best_for" styles/*.yaml`. Treat a playbook's `quality_rules` as constraints. For custom, atelier, brand, launch, or hero work, read `skills/meta/taste-direction.md` before choosing one and carry its `taste_profile` into the proposal so later stages preserve the design read.

Choosing an animation runtime (Remotion primitives, GSAP, framer-motion, Lottie, Manim, D3): `skills/meta/animation-runtime-selector.md`.

### Hand-drawn doodle animation: Ink Theater / Ink Puppet

For a hand-drawn ink look ("a sketch that comes to life", a stick figure that walks or dances, whiteboard-doodle explainers), use the **Ink Theater** engine and **Ink Puppet** mocap (`skills/creative/ink-theater.md`, `ink-theater/README.md`). It is a style plus engine, not a pipeline: illustration and contraption pieces run on `animation`; a mocap character (draws itself → walks / dances / waves via `InkPuppet.choreograph([...])`) runs on `character-animation`. Entry points: **`/ink-art`** (a vector doodle from scratch) and **`/animated-drawing`** (animate a *supplied* drawing, raster; `skills/creative/animated-drawing.md`). Character motion comes only from named mocap clips.
