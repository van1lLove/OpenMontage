# Planning and Music Plan

Reached from [`AGENT_GUIDE.md`](../../AGENT_GUIDE.md) at the proposal / idea stage, before committing to execution.

## User-Facing Planning Protocol

Present, before any asset generation:

1. `4-5` concept directions when the brief is still open.
2. Recommended pipeline.
3. Recommended tool path.
4. Alternative tool paths that are actually available.
5. Cost estimate and quality tradeoffs.
6. **Music plan** — every pipeline that has audio (below).
7. Render runtime and composition authoring mode (`AGENT_GUIDE.md` → "Present Both Composition Runtimes (HARD RULE)"; [`composition.md`](composition.md)).
8. Production plan by stage.
9. Approval gate: asset generation starts only after the user approves this plan.

When the user prefers a specific vendor and that tool is available, surface it directly; provider choice is always visible to the user.

## Music Plan (Mandatory)

Surface the music situation at proposal/idea time, where a gap is cheap to fix, rather than at the asset stage.

Check sources in this order:

1. **User music library** (`music_library/`, gitignored, where users drop royalty-free tracks): if it exists and has tracks, list them with durations and let the user pick.
2. **Music generation APIs**: `registry.get_by_capability("music_generation")`. Report status honestly, including quota if known.
3. **Royalty-free sources**: the user can supply a track (YouTube Audio Library, Jamendo, other free sources) via the `music_library/` drop path.

Always offer explicit choices:

- use a track from their library (which one?),
- provide a different track (drop it in `music_library/`),
- generate one via API (if available — name the provider and cost),
- proceed without music.

If no music source is available, say so explicitly at this stage.

Record the music decision in the proposal/brief artifact so the asset director knows what to do.
