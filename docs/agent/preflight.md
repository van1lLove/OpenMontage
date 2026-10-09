# Preflight and Provider Menu

Reached from [`AGENTS.md`](../../AGENTS.md) → Rule Zero at the start of every production run, before any creative work.

## Mandatory Preflight

```bash
python -c "from tools.tool_registry import registry; import json; registry.discover(); print(json.dumps(registry.provider_menu_summary(), indent=2))"
```

The summary is the human-ready rollup; the raw `registry.support_envelope()` is megabytes of JSON on a well-configured machine, so keep it out of chat and use it only for debugging. `registry.provider_menu()` gives per-tool detail the summary collapses. Translate its four fields:

- `composition_runtimes`: booleans for `ffmpeg`, `remotion`, `hyperframes`; the source of truth for `AGENTS.md` → "Present Both Composition Runtimes (HARD RULE)".
- `capabilities[]`: `configured / total` counts and providers per capability family, ready for the "N of M configured" menu.
- `setup_offers[]`: unavailable tools a 1-minute env var would unlock. Lead with these.
- `runtime_warnings[]`: e.g. "hyperframes: npm package not resolvable". Relay them verbatim: they are the silent-failure bugs.

Then check every `required_tools` entry of the manifest against the registry, check `fallback_tools` for the unavailable ones, and report `passed`, `degraded`, or `blocked`.

## Provider Menu (Mandatory at Preflight)

Present the summary grouped by capability, not as a flat tool list:

```
YOUR CAPABILITIES

  Video Generation:  0/13 configured
  Image Generation:  1/7 configured
  Text-to-Speech:    1/3 configured
  Composition:       3/3 configured (FFmpeg, video_stitch, video_trimmer)

  You can produce videos now with images + TTS + FFmpeg.

QUICK SETUP OPTIONS (1-minute each, an env var in .env)
  Video Generation: FAL_KEY unlocks 3 providers   <- group tools sharing an env var
LOCAL OPTIONS (free, needs hardware)
  Tools with runtime LOCAL or LOCAL_GPU
ALREADY AVAILABLE
  What works now
```

- Read provider names, env var names, and setup URLs from each tool's `install_instructions` and `dependencies` every time; hardcoded values drift between releases.
- Show what the user CAN do now, then what they COULD unlock.
- If the user declines setup, proceed with the best available path and drop the topic.

## Setup Offer Protocol

When a tool is `UNAVAILABLE` but fixable, offer setup help rather than working around it silently:

| Fix | Action |
|-----|--------|
| **1-minute** (env var) | Offer to configure it now from `install_instructions` |
| **5-minute** (install) | Explain what to install and why |
| **Complex** (GPU, model download) | Note the limitation and what it would unlock; move on |

Every offer states what the user is missing, what they would gain, and the cost difference (free local vs paid API).

## Capability Discovery

Route multi-provider capabilities through the selectors (`tts_selector`, `video_selector`, `image_selector`); the legacy names `tts_cloud`, `tts_engine`, `video_gen` no longer exist. The manifest's `tools_available` is the authoritative per-stage tool list. Tool strengths come from registry fields (`best_for`, `runtime`, `status`, `install_instructions`), never from memory: `registry.capability_catalog()` groups tools by capability, `registry.provider_catalog()` by vendor, and a tool's `usage_location` says how it works.
