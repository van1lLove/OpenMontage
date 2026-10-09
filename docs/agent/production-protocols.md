# Production Protocols

Reached from [`AGENT_GUIDE.md`](../../AGENT_GUIDE.md) when initializing a project workspace, writing a checkpoint, or validating a stage artifact. Step-by-step checkpoint mechanics (resume, partial progress, approval handling) live in `skills/meta/checkpoint-protocol.md`.

## Project Workspace

Every run writes into `projects/<project-id>/` (gitignored; all generated assets are regenerable). Name the project in kebab-case from the video title (`hidden-math-of-nature`, `how-music-rewires-brain`).

```
projects/<project-name>/
├── artifacts/          # JSON artifacts from each stage (research_brief, script, scene_plan, etc.)
├── assets/
│   ├── images/         # Generated images (PNG)
│   ├── video/          # Generated video clips (MP4)
│   ├── audio/          # Narration segments + final mix (MP3/WAV)
│   ├── music/          # Background music track (MP3)
│   └── subtitles.srt   # Generated subtitles
├── history/            # Superseded checkpoints, archived automatically
└── renders/
    └── final.mp4       # Final rendered video (the deliverable)
```

Atelier and HyperFrames-skill runs follow the same contract: hand-authored compositions still write the canonical artifacts they have (script or beats-plan, scene_plan-equivalent, asset manifest) plus checkpoints here. The Backlot board is runtime-agnostic; runs that skip the artifacts get a degraded board.

## Stage Contract

- Each stage produces one canonical artifact that validates against its JSON schema in `schemas/artifacts/`.
- Non-canonical outputs such as media files go in stage-specific directories.
- Tools record seeds and model versions for reproducibility.

## Checkpoint Rules

Agents coordinate through canonical JSON artifacts, checkpoints, pipeline manifests, and the tool registry. Checkpoint schema: `schemas/checkpoints/checkpoint.schema.json`.

- Checkpoints live at `projects/<project_id>/checkpoint_<stage>.json`; the Backlot board watches them.
- `status` is one of `completed`, `failed`, `awaiting_human`, `in_progress`.
- Write an `in_progress` checkpoint on entering each stage. During `assets`/`compose`, refresh `metadata.partial_progress` after each completed scene/asset unit; this powers live progress on the board.
- `completed` and `awaiting_human` checkpoints include the stage's canonical artifact.
- Superseded checkpoints are archived to `projects/<project_id>/history/`, so stage re-runs keep run history.
- Invalid checkpoints or invalid canonical artifacts are contract violations and fail fast.

## Pipeline Manifests

Pipelines are declarative YAML manifests in `pipeline_defs/`. Each stage declares `skill` (director skill path), `produces`, `tools_available`, `review_focus`, `success_criteria`, `human_approval_default`.
