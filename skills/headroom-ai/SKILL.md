---
name: headroom-ai
description: "Shrink LLM context before each model call with Headroom (pip package `headroom-ai`). Compresses tool outputs, logs, JSON and chat history in-process, no API key. Use when an agent loop or script sends large ffmpeg/ffprobe logs, transcript JSON, render output, or long chat history to Claude and is hitting token limits or cost."
version: 1.0.0
---

# Headroom: context compression

Headroom sits between your message list and the API call. It keeps the signal (errors, recent turns, the user's actual question) and collapses repetition (thousands of identical `INFO` lines, uniform JSON arrays). Compressed blocks leave a retrieval marker (`hash=...`) so nothing is silently lost.

Tested against `headroom-ai` 0.44.0.

## Install

Optional dependency. Install only when a session needs it:

```bash
pip install headroom-ai          # or: pip install -e ".[context]" inside this repo
headroom --version               # CLI is `headroom`, not `headroom-ai`
```

## Library usage

```python
import anthropic
from headroom import compress

client = anthropic.Anthropic()

result = compress(messages, model="claude-opus-5-5")   # Anthropic or OpenAI message format

response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=4096,
    messages=result.messages,
)

print(f"{result.tokens_before} -> {result.tokens_after} tokens "
      f"({result.compression_ratio:.0%} saved) via {result.transforms_applied}")
```

`compress()` returns a `CompressResult`: `messages`, `tokens_before`, `tokens_after`, `tokens_saved`, `compression_ratio`, `transforms_applied`, `lossless`, `reverted_messages`, `diagnostics`.

### Tuning (keyword args, or `config=CompressConfig(...)`)

| Arg | Default | Use |
|-----|---------|-----|
| `protect_recent` | `4` | Last N messages left untouched. |
| `compress_user_messages` | `False` | Set `True` when the user pasted a big document. |
| `compress_system_messages` | `True` | |
| `target_ratio` | `None` | e.g. `0.5` to keep about half. |
| `min_tokens_to_compress` | `250` | Smaller blocks pass through. |
| `frozen_message_count` | `0` | Freeze a prefix so prompt caching still hits. |
| `verify_lossless` | `False` | Check before trusting a lossless claim. |
| `optimize` | `True` | `False` = passthrough, for A/B comparisons. |
| `model_limit` | `200000` | Context window used for budgeting. |

## Where it helps in video-use

- **ffmpeg / ffprobe logs** in tool results. Repetitive frame lines collapse and `ERROR` lines survive. In a smoke test, an 801-line ffmpeg-style log dropped from 10,530 to 117 tokens with the error kept.
- **Transcript JSON** from `helpers/transcribe.py` before it goes to a model. Prefer `pack_transcripts.py` first: it's purpose-built. Use Headroom on whatever is still large after that.
- **Long editing sessions** where chat history plus tool output pushes toward the context limit.

## Rules

- Compress right before the API call. Keep the original `messages` as your source of truth and don't write compressed text back to disk.
- Never compress EDL, timestamps, or cut lists that a model must reproduce exactly. Put them in the protected recent window or send them uncompressed.
- If an answer looks wrong, rerun with `optimize=False` to check whether compression caused it.

## Proxy / CLI (optional)

`headroom proxy` runs an optimizing proxy so existing clients get compression without code changes. Other subcommands: `headroom doctor`, `headroom dashboard`, `headroom inspect`. Run `headroom --help` for the rest. Docs: https://docs.headroomlabs.ai
