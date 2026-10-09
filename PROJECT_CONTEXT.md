# OpenMontage - Project Context

Architecture and key files. Code conventions (adding tools or pipelines, naming, selectors, tests) live in [`CODING_STANDARDS.md`](CODING_STANDARDS.md).

## Architecture: Instruction-Driven (Agent-First)

The AI agent IS the intelligence. Python exists only for tools and persistence. Everything else — orchestration, creative decisions, review, stage transitions — lives in instructions (YAML manifests + markdown skills) the agent follows.

```
Agent reads pipeline manifest (YAML) → reads stage director skill (MD)
→ uses tools (Python BaseTool) → self-reviews (meta skill)
→ checkpoints (Python utility) → presents to human for approval
```

## Source of Truth

- **Agent guide & contract:** `AGENT_GUIDE.md` (request routing, hard rules, stage agents, protocols, available pipelines); branch reference in `docs/agent/`
- **Code conventions:** `CODING_STANDARDS.md`
- **Skill index:** `skills/INDEX.md`
- **Tool registry:** `tools/tool_registry.py`
- **Pipeline manifests:** `pipeline_defs/`
- **Artifact schemas:** `schemas/artifacts/`
- **Style playbooks:** `styles/*.yaml` (schema: `schemas/styles/playbook.schema.json`)
- **Stage director skills:** `skills/pipelines/<pipeline>/<stage>-director.md`
- **Meta skills:** `skills/meta/*.md` (reviewer, checkpoint-protocol, skill-creator)
- **Architecture deep-dive:** `docs/ARCHITECTURE.md`

## Knowledge Architecture (3 Layers)

See `docs/agent/layers-and-styles.md` → "Layer Map". Each tool's `agent_skills[]` field bridges Layer 1 → Layer 3; `skills/INDEX.md` has the full mapping.

## Key Patterns

- **Pipeline state machine:** `idea -> script -> scene_plan -> assets -> edit -> compose -> publish`
- **Instruction-driven stages:** Each stage has a director skill (MD) that teaches the agent HOW
- **Pipeline manifests:** Declarative YAML defining stages, skills, tools, review focus, approval gates
- **Capability-first tool design:** selector tool plus explicit provider tools per family (see `CODING_STANDARDS.md`)
- **Style playbooks:** YAML defining visual language, typography, motion, audio, asset generation constraints
- **Artifacts are canonical:** `brief`, `script`, `scene_plan`, `asset_manifest`, `edit_decisions`, `render_report`, `publish_log`
- **Checkpoint policy** lives in pipeline manifest (`human_approval_default` per stage) + `skills/meta/checkpoint-protocol.md`
- **Cost tracker** (`tools/cost_tracker.py`) manages budget: estimate -> reserve -> reconcile

## Key Files

| File | Purpose |
|------|---------|
| `config.yaml` | Global configuration |
| `lib/config_model.py` | Runtime config loader (Pydantic) |
| `lib/checkpoint.py` | Checkpoint writer/reader |
| `lib/pipeline_loader.py` | Pipeline manifest loader + helpers |
| `lib/media_profiles.py` | Platform-specific render profiles |
| `styles/playbook_loader.py` | Style playbook loader + validator + design intelligence (color/type/a11y) |
| `tools/base_tool.py` | ToolContract base class |
| `tools/tool_registry.py` | Tool discovery and reporting |
| `tools/cost_tracker.py` | Budget governance |
| `tools/video/video_stitch.py` | Multi-clip assembly (stitch, spatial, validate, preview) |
| `tools/video/video_compose.py` | Runtime-aware composition orchestrator — routes to Remotion / HyperFrames / FFmpeg based on `edit_decisions.render_runtime` |
| `tools/video/hyperframes_compose.py` | HyperFrames runtime — workspace materialization, `hyperframes lint`/`validate`/`render`, FFmpeg floor check |
| `tools/character/character_animation.py` | Local character-animation tools — character specs, SVG rig plans, pose libraries, action timelines, HyperFrames packages, and QA reports |
| `lib/hyperframes_style_bridge.py` | Playbook → CSS custom properties + `DESIGN.md` bridge for HyperFrames workspaces |
| `remotion-composer/src/components/` | 8 Remotion components (TextCard, StatCard, ProgressBar, CalloutBox, ComparisonCard + charts/) |
| `.agents/skills/hyperframes*/` | Vendored HyperFrames Layer 3 skills (authoring contract, CLI, registry, website-to-video) |
| `skills/core/hyperframes.md` | Layer 2 — when OpenMontage should pick HyperFrames vs Remotion, artifact → workspace mapping |
| `schemas/styles/playbook.schema.json` | Playbook schema v2 with design tokens (chart_palette, scale_system, weight_matrix, color_rules) |
| `tests/qa/` | Quality validation test scripts for tool-by-tool output inspection |

Available pipelines: `pipeline_defs/` (one manifest per pipeline); `AGENT_GUIDE.md` → "Available Pipelines" has best-for and stability.
