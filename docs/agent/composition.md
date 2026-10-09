# Composition: Runtimes and Authoring Mode

Reached from [`AGENT_GUIDE.md`](../../AGENT_GUIDE.md) when designing the visual approach at proposal, and whenever a stage touches `video_compose`. The runtime-selection rule and the runtime lock live in `AGENT_GUIDE.md` → "Present Both Composition Runtimes (HARD RULE)"; this file is the reference behind them.

## Composition Runtimes (Inside video_compose)

`video_compose` has three parallel, unranked render engines. The choice is made at proposal (`proposal_packet.production_plan.render_runtime`) and carried through `edit_decisions.render_runtime` unchanged; `video_compose` dispatches on that field (`_render_via_hyperframes`, `_remotion_render`, `_render_via_ffmpeg`).

Check availability:

```bash
python -c "
from tools.tool_registry import registry
registry.discover()
info = registry._tools['video_compose'].get_info()
print('Render engines:', info.get('render_engines'))
print('Remotion note:', info.get('remotion_note'))
print('HyperFrames note:', info.get('hyperframes_note'))
"
```

| Engine | Used For | Requires |
|--------|----------|----------|
| **FFmpeg** | Video-only cuts, concat, trim, subtitle burn | `ffmpeg` binary (always available) |
| **Remotion** | React-based composition: still images → animated video, text cards, stat cards, charts, callouts, comparisons, transitions with spring physics, word-level caption burn, TalkingHead avatar | Node.js (`npx`) + `remotion-composer/` + `node_modules` |
| **HyperFrames** | HTML/CSS/GSAP composition: kinetic typography, product promos, launch reels, website-to-video, registry-block-driven scenes, SVG character rigs | Node.js ≥ 22 + FFmpeg + `npx` (consumed via `npx hyperframes`) |

Pick the runtime that fits the concept: HTML/GSAP for motion-graphics-heavy concepts it expresses naturally, Remotion for briefs that reuse the existing React scene stack. `skills/core/hyperframes.md` holds the Remotion-vs-HyperFrames decision matrix.

### Designing around Remotion

When Remotion is available, plan production around it:

- Explainers with the `flat-motion-graphics` playbook → Remotion animated scenes rather than Ken Burns.
- Data-driven videos → Remotion stat cards and charts rather than static screenshots.
- Pipelines using still images → Remotion spring animations rather than FFmpeg pan-and-zoom.
- Screen demos of a CLI/terminal/install flow → `TerminalScene` (synthetic screen recording; `.agents/skills/synthetic-screen-recording/SKILL.md`): faster, deterministic, privacy-safe. Use real capture (`screen_recorder`, `cap_recorder`, `playwright-recording`) only for a real app UI or unpredictable live behavior.

When Remotion is unavailable and `render_runtime="remotion"` was not locked, `video_compose` may use FFmpeg Ken Burns motion on stills: it works but is less engaging, so name that tradeoff in the proposal.

### Remotion scene types

`remotion-composer/SCENE_TYPES.md` is the authoritative list with cut schemas. Scene types usable via `cut.type`: `text_card`, `stat_card`, `callout`, `comparison`, `hero_title`, `terminal_scene`, `anime_scene`, `bar_chart`, `line_chart`, `pie_chart`, `kpi_grid`, `progress_bar`. Overlay types: `section_title`, `stat_reveal`, `hero_title`, `provider_chip`.

These stock scene types are the **templated** path (below).

## Composition Authoring Mode — Templated vs Atelier

Orthogonal to *runtime* is *authoring mode*: how the composition is built. Present it as its own proposal decision and log it in `decision_log` (`category: "composition_mode"`).

- **Templated** — assemble the stock `cut.type` scene types into the `Explainer`/`CinematicRenderer` compositions. Fast, cheap, reliable, and the reason most videos look alike. Right for batch output, localization variants, quick drafts, and low-stakes internal clips.
- **Atelier** — hand-author the composition from scratch: bespoke scenes, a one-off theme, and motion written for this piece, rendered via `composition_mode: "atelier"` (`video_compose` → `_render_via_atelier`). A fresh visual language every time.

**Default to atelier for hero work**: marketing, launches, brand pieces, any single-deliverable explainer that must impress. The deciding rule: *reuse engine knowledge, never creative components.* In atelier mode the stock scene-type catalog, `hyperframes-registry` blocks, fixtures, and finished components serve only as a *mechanics codex* to read, never as parts to assemble: they are frozen looks that reintroduce sameness.

Atelier sequence:

1. `skills/meta/taste-direction.md` — set the design read and taste dials.
2. `skills/meta/bespoke-composition.md` — art direction (`visual-style`) → motion principles (Disney 12 via `framer-motion`/`lottie-bodymovin`) → engine mechanics (`remotion-best-practices`, stock components as codex) → render via the atelier path.
3. Close with a **distinctness review**: *could this be any other product's video? does it reuse a look I've made before?*

Atelier costs more tokens and iteration than templated; say so at proposal so the user opts in knowingly.
