# OpenMontage - Coding Standards

Conventions for changing OpenMontage's code: tools, pipelines, and tests. For running a video production, follow [`AGENT_GUIDE.md`](AGENT_GUIDE.md) instead.

## Where Logic Lives

Python = tools + persistence. Orchestration, creative decisions, review logic, and checkpoint policy live in instructions (pipeline manifests + director skills + meta skills), never in Python: no Python orchestrator, reviewer, or handlers.

## Adding a Tool

1. Inherit from `BaseTool` in `tools/base_tool.py` (every production tool does).
2. Put the tool in the correct capability package (`tools/audio/`, `tools/video/`, `tools/enhancement/`, `tools/analysis/`, `tools/graphics/`, `tools/avatar/`, `tools/subtitle/`).
3. Prefer the selector-plus-provider pattern (see Selector Pattern below):
   - one capability router tool for agent convenience
   - one concrete tool per real provider/runtime path
   - Example: `tts_selector` + `elevenlabs_tts` / `google_tts` / `openai_tts` / `piper_tts`
4. Set all contract fields (name, version, tier, capability, provider, supports, fallback_tools, agent_skills, etc.). Support-envelope reporting is the source of truth for capability, status, and resource requirements.
5. Implement `execute()` returning a `ToolResult`.
6. Let discovery happen through `tools/tool_registry.py`; do not depend on ad hoc imports.
7. Add a JSON schema in `schemas/tools/` if the tool has complex I/O.
8. Add tests only after the runtime path is correct.

### Tool Class Naming Convention

All tool classes use **PascalCase without a "Tool" suffix**:

| Module | Class Name | NOT |
|--------|-----------|-----|
| `tools.audio.music_gen` | `MusicGen` | ~~MusicGenTool~~ |
| `tools.video.video_compose` | `VideoCompose` | ~~VideoComposeTool~~ |
| `tools.audio.audio_mixer` | `AudioMixer` | ~~AudioMixerTool~~ |
| `tools.tts.elevenlabs_tts` | `ElevenLabsTTS` | ~~ElevenLabsTTSTool~~ |
| `tools.analysis.transcriber` | `Transcriber` | ~~TranscriberTool~~ |
| `tools.subtitle.subtitle_gen` | `SubtitleGen` | ~~SubtitleGenTool~~ |

When in doubt, check: `grep "^class " tools/<path>.py`

All tools are called via `.execute(params_dict)` (returns `ToolResult` with `.success`, `.data`, `.error`), NOT `.run()`.

### Selector Pattern

Three selector tools abstract multi-provider capabilities. **Selectors auto-discover providers from the registry**, so adding a new provider tool makes it available through the selector with no selector code changes.

| Selector | Routes to | How it discovers |
|----------|-----------|-----------------|
| `tts_selector` | All tools with `capability="tts"` | `registry.get_by_capability("tts")` |
| `image_selector` | All tools with `capability="image_generation"` | `registry.get_by_capability("image_generation")` |
| `video_selector` | All tools with `capability="video_generation"` | `registry.get_by_capability("video_generation")` |

Selectors route based on: user preference > availability > discovery order. They adapt input schemas between providers transparently.

## Adding a Pipeline

1. Create a YAML manifest in `pipeline_defs/` (validated by `schemas/pipelines/pipeline_manifest.schema.json`).
2. Create stage director skills in `skills/pipelines/<pipeline-name>/` (one per stage, idea through publish).
3. Reference meta skills (reviewer, checkpoint-protocol) in the manifest.
4. Add compatible playbooks to the manifest.
5. Add contract tests in `tests/contracts/`.

## Editing Agent Docs

Contract tests read the agent docs. `tests/contracts/test_phase0_contracts.py` asserts `AGENT_GUIDE.md` keeps the headings `## Orchestrator`, `## Stage Agents`, `## Reviewer Protocol`, `## Communication Protocol`, `## Human Checkpoint Protocol`, and that every platform wrapper (`AGENTS.md`, `CLAUDE.md`, `CODEX.md`, `CURSOR.md`, `COPILOT.md`) mentions `AGENT_GUIDE.md`. Other tests grep `AGENT_GUIDE.md` for `Present Both Composition Runtimes`, `HARD RULE`, `taste-direction.md`, and `provider_menu_summary`. Skills, schemas, and tool error strings cite `AGENT_GUIDE.md` headings by name; grep the repo before renaming one.
