# OpenMontage - Coding Standards

Conventions for changing OpenMontage's code, tests, and agent docs. Running a video production follows [`AGENTS.md`](AGENTS.md).

## Where Logic Lives

Python = tools + persistence. Orchestration, creative decisions, review logic, and checkpoint policy live in instructions (pipeline manifests, director skills, meta skills): Python has no orchestrator, reviewer, or stage handlers.

Skills teach tool usage; tool source is for debugging, audits, and verifying the governance contract. When a skill and a tool disagree, the source wins: fix the skill so the next agent skips the dive.

## Adding a Tool

1. Subclass `BaseTool` (`tools/base_tool.py`) in the matching capability package under `tools/`.
2. Use the selector-plus-provider pattern: one concrete tool per real provider/runtime path behind a capability router (`tts_selector` → `elevenlabs_tts` / `google_tts` / `piper_tts`). Selectors discover providers via `registry.get_by_capability(...)` and route by user preference > availability > discovery order, so a new provider tool needs no selector change.
3. Fill every contract field (`name`, `version`, `tier`, `capability`, `provider`, `supports`, `fallback_tools`, `agent_skills`, ...). Support-envelope reporting is the source of truth for capability, status, and resource requirements.
4. Implement `execute(params_dict)` returning a `ToolResult` (`.success`, `.data`, `.error`); callers use `.execute()`, never `.run()`.
5. Let `tools/tool_registry.py` discover the tool; import it nowhere by hand.
6. Add a JSON schema in `schemas/tools/` when the I/O is complex.
7. Add tests once the runtime path works.

Class names are PascalCase without a `Tool` suffix: `MusicGen`, `VideoCompose`, `ElevenLabsTTS`. Check an existing one with `grep "^class " tools/<path>.py`.

## Adding a Pipeline

1. Write `pipeline_defs/<name>.yaml` (validated by `schemas/pipelines/pipeline_manifest.schema.json`), listing meta skills (reviewer, checkpoint-protocol) and compatible playbooks.
2. Write one director skill per stage in `skills/pipelines/<name>/`.
3. Add contract tests in `tests/contracts/`.

## Editing Agent Docs

`AGENTS.md` is the single always-loaded entry (`CLAUDE.md` imports it; the other platform files point to it): routing, Rule Zero, and the hard rules every production run needs. Reference only some branches reach goes in `docs/agent/`, behind a pointer in `AGENTS.md` that names the triggering branch. Keep each rule in one file; replace anything a one-command lookup answers (`pipeline_defs/`, `styles/`, registry output, `skills/INDEX.md`) with that command.

Tests read these docs. `tests/contracts/test_phase0_contracts.py` asserts `AGENTS.md` keeps `## Orchestrator`, `## Stage Agents`, `## Reviewer Protocol`, `## Communication Protocol`, `## Human Checkpoint Protocol`, and that every platform wrapper mentions `AGENTS.md`. Other tests grep `AGENTS.md` for `Present Both Composition Runtimes`, `HARD RULE`, `silently`, `taste-direction.md`, and `provider_menu_summary`. Skills, schemas, and tool error strings cite headings by name: `AGENTS.md` → "Present Both Composition Runtimes (HARD RULE)", "Escalate Blockers Explicitly" (also in a `video_compose` error string), "Decision Communication Contract", "Rule Zero"; `docs/agent/preflight.md` → "Mandatory Preflight", "Provider Menu (Mandatory at Preflight)", "Setup Offer Protocol"; `docs/agent/composition.md` → "Composition Authoring Mode". Grep the repo before renaming or moving one.
