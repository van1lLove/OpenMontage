# OpenMontage

Agent contract for running video productions. Architecture: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Route the Request

| Branch | Signal | First action |
|--------|--------|--------------|
| **Onboarding** | Vague or exploratory ("make me a video", "what can you do?") | Read `skills/meta/onboarding.md` |
| **Reference video** | A video offered as *inspiration* ("make something like this") | [Reference Video Entry Point](#reference-video-entry-point) |
| **Production** | A specific video request, or footage to edit ("cut this into clips") | [Rule Zero](#rule-zero--all-production-goes-through-a-pipeline) |
| **Code / docs** | Changing tools, pipelines, tests, or these agent docs | Read [`CODING_STANDARDS.md`](CODING_STANDARDS.md) |

### Reference Video Entry Point

A reference video is a creative brief. Run the analysis workflow in `skills/meta/video-reference-analyst.md` with the local tools, summarize what makes the reference work (content, pacing, structure, style), then run Rule Zero and pitch **2-3 differentiated concepts** for the user's own version. Footage the user wants *edited* is the production branch: `source_media_review` plus a footage-led pipeline.

## Rule Zero — All Production Goes Through a Pipeline

Every request to make video content runs through the pipeline system:

1. **Pick the pipeline**: `grep -E -A2 "^(description|stability):" pipeline_defs/*.yaml`. Ask the user when unclear; tell them when they pick a `beta` one (rough edges).
2. **Read its manifest** `pipeline_defs/<pipeline>.yaml`: stages, tools, review focus, approval gates.
3. **Preflight**: run `registry.provider_menu_summary()`, present it as a capability menu ("X of Y configured", what works now, what one env var unlocks), and report `passed`, `degraded`, or `blocked`. Creative work starts once the user knows the real capability envelope. How: [`docs/agent/preflight.md`](docs/agent/preflight.md).
4. **Run the [Orchestrator](#orchestrator)** stage by stage.
5. **Read Layer 3 skills before every generation call.** Each tool's `agent_skills` names its vendor skills in `.agents/skills/` (provider prompting, parameters, quality technique). `skills/INDEX.md` maps the three layers: tools → OpenMontage skills → vendor skills.

Hard guardrail: tools run only inside a pipeline stage, after its director skill and the tool's Layer 3 skill are read, with preflight, checkpoints, and review intact. Ad-hoc scripts and raw API calls that bypass the pipeline are out of bounds.

## Orchestrator

The agent runs the production state machine; the manifest sets the stage order.

Before the first stage:

1. `python -c "from lib.checkpoint import init_project; init_project('<project-id>', title='<Title>', pipeline_type='<pipeline>')"` creates `projects/<project-id>/` (kebab-case id from the title). Every output of the run goes under it via an explicit `output_path`, atelier and HyperFrames-skill runs included: it is the only place the user's board sees.
2. `python -m backlot open <project-id>` opens the board. The board is an observer: if it fails, continue.

Per stage: `checkpoint.get_next_stage()` → read `skills/pipelines/<pipeline>/<stage>-director.md` → work with `tools/` → [review](#reviewer-protocol) → [checkpoint](#communication-protocol) → pause where the manifest [gates](#human-checkpoint-protocol) the stage.

## Decision Communication Contract

The user knows which provider, model, and render path is in play before it runs.

- **Announce** before any paid or consequential generation call: tool, provider, model or variant, why, and sample vs batch.
- **Ask before major changes**: provider or model family, video-led → still-led, composition engine, dropping narration, music, or another approved element, sample → batch. Prompt refinements inside an approved path need no approval unless they shift the creative direction.
- **Re-log changed decisions** (binding, every stage). `decision_log` is append-only, and the board's Decisions rail keys each decision on (`category`, `subject`), showing the latest entry as current. When a logged choice changes, append an entry with the same `category` AND `subject`, moving the superseded option into `options_considered` with `rejected_because`. Editing only a downstream artifact leaves the board stale; rewording the subject splits one decision in two.

### Present Both Composition Runtimes (HARD RULE)

When both Remotion and HyperFrames are available (`provider_menu_summary()["composition_runtimes"]`), present **both** before locking `render_runtime` at proposal, in every pipeline that invokes `video_compose`. Never pick one silently, even when a manifest or director skill names a default: that is input to the conversation, not a decision. Per runtime, give what it is best at for this brief and one honest tradeoff, then recommend one, tied to the brief's `delivery_promise`. Wait for approval and log `render_runtime_selection` with BOTH runtimes (plus any `ffmpeg` option) in `options_considered`; a single-option entry when both were available is a CRITICAL reviewer finding. With only one installed, proceed on it, tell the user, and log the missing one with `rejected_because: "runtime not available on this machine"`.

**Runtime lock.** The approved `render_runtime` holds through compose. If it becomes unavailable or fails to render, that is a blocker: [escalate](#escalate-blockers-explicitly), and switch engines only after user approval and a new `render_runtime_selection` entry.

Authoring mode (templated vs atelier) is a separate presented decision (`category: "composition_mode"`); hero work defaults to atelier, starting from `skills/meta/taste-direction.md`. Runtimes, scene types, atelier, style playbooks, ink doodles: [`docs/agent/composition.md`](docs/agent/composition.md).

### Motion-Required Requests

When the promise depends on moving shots (trailers, cinematic teasers, hype edits, avatar videos), motion is a hard requirement. Confirm the runtime and providers up front and deliver motion-led video. A Ken Burns, animatic, slide-based, or still-led cut is a downgrade the user approves explicitly before you spend on it; if clip generation blocks the approved treatment, stop and tell the user.

### Escalate Blockers Explicitly

Surface a blocker immediately: what was attempted, what failed, its class (auth, provider access, tool bug, prompt/design quality), a shortlist of options with tradeoffs, and your recommendation. Prepare alternatives freely; execute one only after the user approves it. That covers provider and model swaps, fallback tools, prompt-only substitutes for reference-driven generation, and still animatics in place of motion. Log every substitution and blocked option in `decision_log`.

## Planning

At proposal, present: concept directions (4-5 when the brief is open), pipeline, recommended and available alternative tool paths, cost and quality tradeoffs, runtime and authoring mode, the music plan, and a production plan by stage. Asset generation starts only after the user approves this plan.

**Music plan** (mandatory for every pipeline with audio). Settle it at proposal, where a gap is cheap. Check in order: the user's `music_library/` (list tracks with durations), `registry.get_by_capability("music_generation")` (name provider, cost, quota), then a track the user drops into `music_library/`. Offer a library track, a supplied track, generation, or no music; when no source exists, say so. Record the choice in the proposal/brief artifact for the asset director.

## Stage Agents

Each stage writes one canonical artifact, the contract for the next: `brief`, `script`, `scene_plan`, `asset_manifest`, `edit_decisions`, `render_report`, validated against `schemas/artifacts/`. Media files go in stage directories; tools record seeds and model versions.

## Reviewer Protocol

Self-review with `skills/meta/reviewer.md` after every stage, before checkpointing. It is advisory and never blocks progression directly.

- Load the stage's `review_focus` from the manifest; apply playbook `quality_rules` as constraints.
- Findings are critical (fix and re-review), suggestion (note and proceed), or nitpick.
- Two review rounds maximum; then pass with warnings and move on.

## Human Checkpoint Protocol

`human_approval_default` in the manifest is binding per stage, and the loaded manifest is the only authority on which stages gate. `lib/checkpoint.py` refuses `completed` on a gated stage without `human_approved=True` (GATE VIOLATION). Gates typically cover idea/proposal, script, scene_plan, **assets** (the user reviews generated assets scene by scene on the Backlot filmstrip before compose locks them in), and publish.

- At a gate: write the checkpoint as `awaiting_human`, present the artifact summary, review findings, and cost snapshot, then **END YOUR TURN**. Further pipeline work in that response is a gate violation.
- Approval is per gate. An early "go ahead" covers only the current gate; full-run pre-authorization counts only when logged in `decision_log` (`category: "approval_policy"`).
- The human approves, requests revision, or aborts.

## Communication Protocol

Agents coordinate through canonical artifacts and checkpoints at `projects/<project_id>/checkpoint_<stage>.json` (schema: `schemas/checkpoints/checkpoint.schema.json`), which the board watches. Write `in_progress` on entering a stage; during `assets`/`compose`, refresh `metadata.partial_progress` after each finished unit for live board progress. `completed` and `awaiting_human` checkpoints carry the stage's artifact. Resume and approval mechanics: `skills/meta/checkpoint-protocol.md`.
