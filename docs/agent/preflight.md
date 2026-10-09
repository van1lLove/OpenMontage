# Preflight and Provider Menu

Reached from [`AGENT_GUIDE.md`](../../AGENT_GUIDE.md) → "Preflight" at the start of every production run, before any creative work.

## Mandatory Preflight

Start with `provider_menu_summary()`: it is the human-ready rollup. The raw `support_envelope()` is a firehose (megabytes of JSON on a well-configured machine); keep it out of chat.

```bash
python -c "
from tools.tool_registry import registry
import json
registry.discover()
print(json.dumps(registry.provider_menu_summary(), indent=2))
"
```

Translate its four fields into plain language:

- `composition_runtimes` — booleans for `ffmpeg`, `remotion`, `hyperframes`. Source of truth for the runtime check in `AGENT_GUIDE.md` → "Present Both Composition Runtimes (HARD RULE)".
- `capabilities[]` — one entry per capability family with `configured / total` counts and provider lists; ready-made for the "N of M configured" menu.
- `setup_offers[]` — unavailable tools whose install is a 1-minute env-var fix. Lead with these when offering upgrades.
- `runtime_warnings[]` — signals like "hyperframes: npm package not resolvable". Relay them to the user verbatim: they are the silent-failure bugs that break the governance contract.

For deeper inspection, only when the summary is not enough:

```bash
# Full menu — grouped available/unavailable per capability.
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.provider_menu(), indent=2))"

# Raw envelope — every tool's full contract. Debugging only.
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.support_envelope(), indent=2))"
```

Then:

1. Read the selected manifest in `pipeline_defs/`.
2. Check every `required_tools` entry against the registry.
3. Check `fallback_tools` for unavailable tools.
4. Report one of: `passed`, `degraded`, or `blocked`.
5. Start production only once the user understands the real capability envelope.

## Provider Menu (Mandatory at Preflight)

Present the `provider_menu_summary()` output as a capability menu grouped by capability, not as a flat tool list. Use `provider_menu()` only for per-tool detail the summary collapses.

```
YOUR CAPABILITIES

  Video Generation:  0/13 configured
  Image Generation:  1/7 configured
  Text-to-Speech:    1/3 configured
  Music Generation:  1/1 configured
  Composition:       3/3 configured (FFmpeg, video_stitch, video_trimmer)

  You can produce videos now with images + TTS + FFmpeg.
  Quick upgrades available — see below.
```

For each capability with unavailable providers, read each tool's `install_instructions` and present setup options grouped by effort:

```
QUICK SETUP OPTIONS (1-minute each — set an env var in .env)

  Video Generation (0/13 -> unlock the biggest upgrade):
    Read install_instructions from each unavailable tool and group by env var.
    Example: if 3 tools need FAL_KEY: "FAL_KEY unlocks 3 providers"

  Image Generation (1/7 -> more style options):
    Same pattern.

LOCAL OPTIONS (free, needs hardware):
  Tools with runtime=LOCAL or runtime=LOCAL_GPU — read from the menu.

Already Available:
  List what's working. The user should feel good about what they have.
```

Rules:

- Read provider names, API key names, and setup URLs from the registry (`install_instructions`, `dependencies`) every time; hardcoded values drift between releases.
- Always show the ratio "X of Y configured" so breadth is visible.
- Show what the user CAN do now, then what they COULD unlock.
- Group tools that share an env var (read the `dependencies` field).
- If the user declines setup, proceed with the best available path and drop the topic.

## Setup Offer Protocol

When a tool is `UNAVAILABLE` but fixable with simple configuration, offer setup help rather than silently working around it.

| Fix Complexity | Action |
|----------------|--------|
| **1-minute fix** (env var) | Offer to help configure now — read `install_instructions` from the tool |
| **5-minute fix** (install) | Explain what to install and why — read `install_instructions` from the tool |
| **Complex fix** (GPU, model download) | Note the limitation, explain what it would unlock, move on |

Every offer states what the user is missing, what they would gain, and the cost difference (free local vs. paid API).

## Capability Discovery

Two layers route capability choice:

- **Selector tools** route at capability level: `tts_selector`, `video_selector`, `image_selector`. Use the selector for multi-provider capabilities inside a stage.
- **Provider tools** call one concrete backend, discovered via the registry.

The pipeline manifest's `tools_available` is the authoritative per-stage tool list. Deleted legacy names (`tts_cloud`, `tts_engine`, `video_gen`) no longer exist; use the selectors.

Query the registry at runtime; tool strengths come from its fields (`best_for`, `install_instructions`, `runtime` LOCAL/API/LOCAL_GPU/HYBRID, `status`), never from memory.

```bash
# Tools grouped by capability (tts, video_generation, image_generation, ...)
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.capability_catalog(), indent=2))"

# Tools grouped by provider (elevenlabs, openai, ffmpeg, ...)
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.provider_catalog(), indent=2))"
```

For finalist tools inspect `capability`, `provider`, `usage_location`, `supports`, `fallback_tools`, `related_skills`.

Capability families:

- **tts** — route via `tts_selector`.
- **video_generation** — cloud, local GPU, stock; route via `video_selector`.
- **image_generation** — cloud, local GPU, stock; route via `image_selector`.
- **music_generation** — music and sound effects.
- **video_post** — composition, stitching, trimming (FFmpeg, always local).
- **audio_processing** — mixing, enhancement (FFmpeg, always local).
- **analysis** — transcription, scene detection, frame sampling.
- **avatar** — talking head and lip sync.
- **character_animation** — character specs, SVG rigs, pose libraries, action timelines, previews, QA.
- **enhancement** — upscale, background removal, face enhance, color grading.

| Question | Where to look |
|----------|---------------|
| What providers serve a capability? | `registry.capability_catalog()` |
| What tools exist for a vendor? | `registry.provider_catalog()` |
| How does a tool actually work? | the tool's `usage_location` |
| Calling a tool class from Python (names, `.execute()`, selector routing)? | [`CODING_STANDARDS.md`](../../CODING_STANDARDS.md) |
