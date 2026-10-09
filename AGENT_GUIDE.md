# OpenMontage - Agent Guide

Operating guide and agent contract for running video productions. Architecture and key files: [`PROJECT_CONTEXT.md`](PROJECT_CONTEXT.md).

## Route the Request

Classify the user's message into one branch, then take that branch's first action.

| Branch | Signal | First action |
|--------|--------|--------------|
| **Vague / onboarding** | Exploratory or "what can you do?" ("make me a video", "help me create something") | Read `skills/meta/onboarding.md` before anything else |
| **Reference video** | A video URL or local file offered as *inspiration* ("make something like this", "use this Reel as a reference") | Follow [Reference Video Entry Point](#reference-video-entry-point) |
| **Production request** | A specific, actionable video request ("a 60-second explainer about black holes"), or source footage to edit ("cut this into clips") | Go straight to [Rule Zero](#rule-zero--all-production-goes-through-a-pipeline) |
| **Code / docs work** | Changing tools, pipelines, tests, or these agent docs | Read [`CODING_STANDARDS.md`](CODING_STANDARDS.md) first |

### Reference Video Entry Point

A reference video is a creative brief, not a web-search or prompt-writing task.

1. Read `skills/meta/video-reference-analyst.md` and run its analysis workflow with the local tools (`video_analyzer`, transcript extraction, scene detection, frame sampling).
2. Produce a grounded summary of the reference: content, pacing, structure, style, what makes it work.
3. Then run Rule Zero (preflight, pipeline selection) and present **2-3 differentiated concepts** for the user's own version rather than a carbon copy.

Source footage the user wants *edited* ("edit this footage", "cut this into clips") is the production branch instead: use `source_media_review` and the matching footage-led pipeline.

## Rule Zero — All Production Goes Through a Pipeline

**Every video production request runs through the pipeline system. No exceptions.** Any request to make, create, produce, or generate video content (trailer, explainer, clip, animation, anything) follows these steps:

1. **Identify the pipeline** from [Available Pipelines](#available-pipelines). If unclear, ask the user.
2. **Read the manifest** `pipeline_defs/<pipeline>.yaml`: stages, tools, quality gates.
3. **Run preflight** and present the capability menu ([Preflight](#preflight)).
4. **Execute stage by stage** via the [Orchestrator](#orchestrator) loop, reading each stage's director skill `skills/pipelines/<pipeline>/<stage>-director.md` before doing any work in that stage.
5. **Read Layer 3 skills before calling tools.** Layer 3 is not optional: every generation tool (video, image, TTS, music) lists its skills in an `agent_skills` field; read them in `.agents/skills/` before writing prompts. They carry provider-specific prompting, parameters, and quality techniques.

Hard guardrail: tools are called only from inside a pipeline stage, after its director skill and the tool's Layer 3 skill are read, with preflight, checkpoints, and review intact. Ad-hoc Python scripts that call tools directly, or raw API calls that skip the pipeline, are out of bounds.

### Available Pipelines

| Pipeline | Best For | Stability |
|----------|----------|-----------|
| `animated-explainer` | Topic to fully generated explainer | production |
| `talking-head` | Footage-led speaker videos | beta |
| `screen-demo` | Screen recordings and walkthroughs | production |
| `clip-factory` | Many clips from one long source | beta |
| `podcast-repurpose` | Podcast highlights and derivatives | beta |
| `cinematic` | Trailer, teaser, and mood-led edits | production |
| `animation` | Motion-graphics and animation-first videos | production |
| `character-animation` | Local rigged cartoon characters and reusable character acting | beta |
| `hybrid` | Source footage plus support visuals | production |
| `avatar-spokesperson` | Presenter-led avatar or lip-sync videos | production |
| `localization-dub` | Subtitle, dub, and translated variants | beta |
| `framework-smoke` | Test: minimal 2-stage smoke test | test |

Beta pipelines are not fully audited: they work with rough edges. Tell the user when they pick one.

### Preflight

Run `registry.provider_menu_summary()` before any creative work and present it as a capability menu ("X of Y configured", what works now, what one env var unlocks). Report preflight as `passed`, `degraded`, or `blocked`, and start production only once the user understands the real capability envelope. Commands, menu format, setup offers, and capability discovery: [`docs/agent/preflight.md`](docs/agent/preflight.md).

## Orchestrator

The agent itself runs the production state machine:

`research -> proposal -> script -> scene_plan -> assets -> edit -> compose`

**Before the first stage:**

1. Initialize the workspace: `python -c "from lib.checkpoint import init_project; init_project('<project-id>', title='<Title>', pipeline_type='<pipeline>')"`. This creates `projects/<project-id>/` and its `project.json` (the marker the Backlot board reads).
2. Open the board: `python -m backlot open <project-id>`. If it fails, continue the production: the board is an observer, never a blocker, and this is the agent's only board duty.

**Per stage:**

1. `checkpoint.get_next_stage()` finds where to resume.
2. Read the stage's director skill to know HOW.
3. Use tools (`tools/`), always passing an explicit `output_path` under `projects/<project-id>/`. That workspace is the only place the user's board can see; atelier and HyperFrames-skill runs write their artifacts and checkpoints there too.
4. Self-review ([Reviewer Protocol](#reviewer-protocol)).
5. Checkpoint ([Communication Protocol](#communication-protocol)).
6. Pause for the human where the manifest gates the stage ([Human Checkpoint Protocol](#human-checkpoint-protocol)).

Workspace layout, naming, and checkpoint mechanics: [`docs/agent/production-protocols.md`](docs/agent/production-protocols.md).

## Decision Communication Contract

The user always knows which provider, model, and render path is in play before it runs.

### Announce Before Execution

Before any paid or consequential generation call, state: the exact tool name, the provider, the model or variant, why it was chosen, and whether it is a sample or a batch run.

### Ask Before Major Changes

Get the user's approval before changing a major production choice:

- provider, or model family / provider variant,
- video-led to still-led treatment,
- composition engine, when that changes the output character,
- dropping narration, music, or another approved creative element,
- sample mode to batch mode.

Minor prompt refinements inside an approved provider/model path need no separate approval unless they materially change the creative direction.

### Re-log Changed Decisions (Binding)

`decision_log` is the board's Decisions rail and the run's audit trail: append-only history. When a logged choice changes mid-run (the user swaps the voice; you switch provider, model, runtime, or music; a fallback overrides an earlier pick), **append a new entry with the same `category` AND the same `subject`** (e.g. `category: "voice_selection"`, `subject: "Narration TTS provider"`), moving the superseded option into `options_considered` with `rejected_because` noting the change. This applies at every stage.

The board keys a decision on its (category, subject) pair and renders the latest entry as current, tagged "revised". Editing only a downstream artifact (`asset_manifest`, a prop) leaves the board showing the stale choice; rewording the subject shows two decisions. The pair is the key because one category can hold distinct decisions (TTS vs image `provider_selection`).

### Present Both Composition Runtimes (HARD RULE)

When both Remotion and HyperFrames are available (`provider_menu_summary()["composition_runtimes"]`, or `video_compose.get_info()["render_engines"]`), present **both** to the user before locking `render_runtime` at proposal. You may recommend one, but never pick a "default" silently, even when the manifest or a director skill suggests one: that suggestion is input to the conversation, not a decision. This applies to every pipeline that invokes `video_compose`.

For each runtime give:

1. A one-sentence plain-language description of what it is best at for **this specific brief**.
2. A one-sentence honest tradeoff.
3. Your recommendation and why, tied to the brief's `delivery_promise` and visual approach.

Wait for explicit approval. Log a `render_runtime_selection` decision whose `options_considered` lists BOTH runtimes plus any applicable `ffmpeg` option; a single-option entry when both were available is a CRITICAL reviewer finding.

If only one runtime is installed, proceed with it and say so ("HyperFrames isn't installed on this machine; I'm proceeding with Remotion. Install HyperFrames if you want the alternative."), logging the missing one with `rejected_because: "runtime not available on this machine"`.

**Runtime lock.** The approved `render_runtime` is a commitment through compose. If it becomes unavailable or fails to render (e.g. HyperFrames with Node < 22, missing `ffmpeg`/`npx`, or `hyperframes doctor` issues), that is a blocker: escalate per [Escalate Blockers Explicitly](#escalate-blockers-explicitly), and switch engines (Remotion, HyperFrames, or FFmpeg) only after user approval and a new `render_runtime_selection` entry. Silent runtime swaps are forbidden.

Authoring mode (templated vs atelier) is a separate presented proposal decision, logged as `category: "composition_mode"`; hero work defaults to atelier, which starts from `skills/meta/taste-direction.md`. Runtime engines, Remotion scene types, and authoring mode: [`docs/agent/composition.md`](docs/agent/composition.md).

### Motion-Required Requests

When the deliverable's promise depends on moving shots (sci-fi trailers, cinematic teasers from generated clips, hype edits, avatar or agent videos), motion is a hard requirement:

- Confirm up front that the runtime and providers the planned treatment depends on are available.
- Deliver motion-led video. A Ken Burns teaser, animatic, slide-based video, or FFmpeg-only still-led cut is a downgrade that needs the user's explicit approval as an animatic or proof-of-concept before you spend tokens or time on it.
- If the runtime or provider clip generation fails in a way that blocks the approved treatment, stop and tell the user immediately.

### Escalate Blockers Explicitly

When a blocker occurs, surface it immediately:

1. What was attempted
2. What failed
3. Whether the issue is auth, provider access, tool bug, or prompt/design quality
4. What options exist next: a shortlist with brief tradeoffs
5. Which option you recommend, and why

Investigate and prepare alternatives freely, but execute one only after the user approves it. That covers provider and model swaps, fallback tools, prompt-only substitutes for reference-driven generation, and still-image animatics in place of true motion. Record every substitution and blocked option in `decision_log`; degraded paths stay visible. Any choice you put to the user follows the same shape: shortlist, tradeoffs, one recommendation, wait for approval.

## Planning

At proposal, present concepts, pipeline, tool paths, cost, the **music plan** (mandatory for every pipeline with audio), runtime and authoring mode, and a production plan by stage. Asset generation starts only after the user approves that plan. Full protocol and music-source order: [`docs/agent/planning.md`](docs/agent/planning.md).

## Stage Agents

Each stage produces one canonical artifact, the contract for the next stage; the stage director skill teaches HOW.

| Stage | Director Skill | Canonical output | Core quality bar |
|------|---------------|------------------|------------------|
| `idea` | `*-director.md` | `brief` | Clear hook, target platform, duration, tone, and user intent |
| `script` | `*-director.md` | `script` | Structured sections, valid timing, coherent narration |
| `scene_plan` | `*-director.md` | `scene_plan` | Ordered scenes, timings, asset requirements |
| `assets` | `*-director.md` | `asset_manifest` | Provenance, paths, model/tool metadata, scene linkage |
| `edit` | `*-director.md` | `edit_decisions` | Concrete cuts, overlays, subtitle/music decisions |
| `compose` | `*-director.md` | `render_report` | Output paths, encoding profile, verification notes |

Canonical artifacts validate against `schemas/artifacts/`; see [`docs/agent/production-protocols.md`](docs/agent/production-protocols.md) → "Stage Contract".

## Reviewer Protocol

Self-review with `skills/meta/reviewer.md` after every stage, before checkpointing. It is advisory and never blocks progression directly.

- Load the stage's `review_focus` from the manifest; apply playbook `quality_rules` as constraints.
- Findings are critical (fix and re-review), suggestion (note and proceed), or nitpick.
- Two review rounds maximum; then pass with warnings and move on.

## Human Checkpoint Protocol

`skills/meta/checkpoint-protocol.md` teaches when to pause.

- `human_approval_default` in the manifest is binding per stage; follow it as written. `lib/checkpoint.py` enforces it: a gated stage cannot be written `completed` without `human_approved=True` (GATE VIOLATION).
- Typically gated: `idea`/`proposal`, `script`, `scene_plan`, **`assets`** (review generated assets scene by scene on the Backlot filmstrip before compose locks them in), and `publish` where present. Most pipelines auto-proceed on `edit` and `compose`, but not all (documentary-montage gates `edit`); the loaded manifest is the only authority.
- At a gate: write the checkpoint as `awaiting_human`, present the artifact summary, review findings, and cost snapshot, then **END YOUR TURN**. Any further pipeline work in that response is a gate violation.
- Approval is per gate. An early "go ahead" covers only the current gate; full-run pre-authorization counts only when recorded as a `decision_log` entry (`category: "approval_policy"`).
- Wait for the human to approve, request revision, or abort.

## Communication Protocol

Agents coordinate through canonical JSON artifacts, checkpoints at `projects/<project_id>/checkpoint_<stage>.json`, pipeline manifests, and the tool registry. Write an `in_progress` checkpoint on entering each stage, and during `assets`/`compose` refresh `metadata.partial_progress` after each finished unit so the board shows live progress. Status values, archiving, and validation: [`docs/agent/production-protocols.md`](docs/agent/production-protocols.md) → "Checkpoint Rules".

## Reference

| Read | When |
|------|------|
| [`docs/agent/preflight.md`](docs/agent/preflight.md) | **Preflight**: running the provider menu, offering setup, discovering tools by capability |
| [`docs/agent/planning.md`](docs/agent/planning.md) | **Proposal**: concept directions, cost, music plan |
| [`docs/agent/composition.md`](docs/agent/composition.md) | **Composition**: choosing a runtime, Remotion scene types, templated vs atelier |
| [`docs/agent/production-protocols.md`](docs/agent/production-protocols.md) | **Workspace and checkpoints**: directory layout, checkpoint status rules, manifest fields |
| [`docs/agent/layers-and-styles.md`](docs/agent/layers-and-styles.md) | **Skills and styles**: finding a Layer 3 skill, picking a playbook, ink-doodle work |
| [`CODING_STANDARDS.md`](CODING_STANDARDS.md) | **Code**: adding tools or pipelines, tool class names, editing these docs |
