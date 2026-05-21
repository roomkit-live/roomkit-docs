# PolarGrid Provider

[PolarGrid](https://polargrid.ai) is a Canadian-hosted inference network with regional edges in **Toronto**, **Vancouver**, and **Montreal**. Use it when data residency on Canadian soil matters and the workload only needs LLM/text — voice and tool calling are limited (see below).

## Install

```bash
pip install roomkit[polargrid]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.polargrid import PolarGridAIProvider, PolarGridConfig

provider = PolarGridAIProvider(
    PolarGridConfig(
        api_key="pg_...",          # from the PolarGrid Console
        model="qwen-3.5-9b",       # or qwen-3.5-27b for higher quality
        region=None,               # None = auto-route; pin in production
    )
)

kit = RoomKit()
kit.register_channel(AIChannel("ai", provider=provider))
```

A full runnable example lives at [`examples/polargrid_ai.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/polargrid_ai.py).

## Configuration

| Field | Default | Notes |
|-------|---------|-------|
| `api_key` | _(required)_ | `pg_...` Bearer token from the PolarGrid Console |
| `model` | `qwen-3.5-9b` | Also `qwen-3.5-27b` (higher quality). Call `list_models()` on the raw SDK for the live catalog. |
| `region` | `None` | `toronto` / `vancouver` / `montreal` — or the IDs `yto-01` / `yvr-02` / `yul-01`. `None` auto-routes. |
| `max_tokens` | `None` | API cap is 4096. |
| `temperature` | `0.7` | 0.0-2.0 |
| `top_p` | `0.9` | 0.0-1.0 |
| `timeout` | `30.0` | Seconds. |
| `max_retries` | `0` | Defaults to 0 so RoomKit's `RetryPolicy` controls retries. |
| `debug` | `False` | Verbose SDK logging. |

## Regions and data residency

PolarGrid runs three Canadian edges. Two ways to choose one:

```python
# Auto-routing — discovers the fastest edge on first call.
config = PolarGridConfig(api_key="pg_...", region=None)

# Pinned — request always goes to the named edge.
config = PolarGridConfig(api_key="pg_...", region="vancouver")
```

The auto-routing path is convenient for development but **pin a region in production** if residency matters. Confirm with PolarGrid whether their auto-routing or failover ever crosses regions before relying on it for compliance.

## Streaming

PolarGrid streams via OpenAI-shaped chunked SSE. RoomKit's provider exposes both plain text deltas and structured events.

```python
async for delta in provider.generate_stream(context):
    print(delta, end="", flush=True)

async for event in provider.generate_structured_stream(context):
    # StreamTextDelta | StreamDone
    ...
```

There are **no thinking deltas** — PolarGrid's chat endpoint doesn't emit a separate reasoning channel today.

## Tool / function calling — not supported (yet)

PolarGrid's chat-completions endpoint does **not** expose tool calling at the time of writing. The provider drops `context.tools` with a warning so the degradation is visible in logs:

```
WARNING roomkit.providers.polargrid: PolarGrid does not support tool/function
calling; 2 tool(s) will be ignored.
```

If you need tools today, route tool-bearing requests to a different provider (Anthropic, OpenAI, Ollama) and keep PolarGrid for text-only generation.

## Vision

`supports_vision` is `False`. The current model catalog (`qwen-3.5-9b`, `qwen-3.5-27b`, `kokoro-82m`, `whisper-large-v3-turbo`) has no multimodal entry on the chat endpoint.

## Error handling

The provider maps the PolarGrid SDK's exception hierarchy onto RoomKit's `ProviderError`:

| PolarGrid exception | `retryable` |
|---------------------|-------------|
| `AuthenticationError` | `False` |
| `ValidationError` | `False` |
| `NotFoundError` | `False` |
| `RateLimitError` | `True` |
| `NetworkError` | `True` |
| `TimeoutError` | `True` |
| `ServerError` | `True` |
| _unknown_ | `True` (let `RetryPolicy` decide) |

## Roadmap

PolarGrid also exposes Speech-to-Text (`whisper-large-v3-turbo`) and Text-to-Speech (`kokoro-82m`) endpoints. RoomKit STT/TTS provider wrappers are planned. There is no speech-to-speech / realtime duplex endpoint today, so `RealtimeVoiceChannel` is out of scope for this provider.
