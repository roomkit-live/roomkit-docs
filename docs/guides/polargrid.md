# PolarGrid Provider

[PolarGrid](https://polargrid.ai) is a Canadian-hosted inference network with regional edges in **Toronto**, **Vancouver**, and **Montreal**. Use it when data residency on Canadian soil matters. It serves OpenAI-shaped chat completions with streaming and tool / function calling; voice is limited (see below).

## Install

```bash
pip install roomkit[polargrid]   # requires polargrid-sdk>=0.10.0
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.polargrid import PolarGridAIProvider, PolarGridConfig

provider = PolarGridAIProvider(
    PolarGridConfig(
        api_key="pg_...",          # from the PolarGrid Console
        model="qwen-3.8-27b",      # default; qwen-3.6-35b-a3b on yul-02
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
| `model` | `qwen-3.8-27b` | The one LLM on the public fleet. `qwen-3.5-27b` was retired on 2026-08-20; `qwen-3.6-35b-a3b` (vision) is a customer pilot with no public edge. See [Models](#models) — or call `PolarGridAIProvider.available_models()` / `list_models()`. |
| `region` | `None` | `toronto` / `vancouver` / `montreal` — or the IDs `yto-01` / `yvr-02` / `yul-01`. `None` auto-routes to an edge already serving the configured model. |
| `max_tokens` | `None` | API cap is 4096. |
| `temperature` | `0.7` | 0.0-2.0 |
| `top_p` | `0.9` | 0.0-1.0 |
| `thinking` | `None` | Toggle qwen reasoning via the `enable_thinking` request flag (sdk 0.8.5+) — `True` on, `False` off, `None` leaves it unset. See [Thinking / reasoning](#thinking-reasoning). |
| `timeout` | `30.0` | Seconds. |
| `connect_timeout` | `5.0` | Seconds, TCP connect alone. `timeout` stays the read budget, so a dead edge is refused in seconds rather than after it. See [Connect vs Read Timeout](production-resilience.md#connect-vs-read-timeout). |
| `max_retries` | `0` | Defaults to 0 so RoomKit's `RetryPolicy` controls retries. |
| `debug` | `False` | Verbose SDK logging. |

## Regions and data residency

PolarGrid runs three Canadian edges. Two ways to choose one:

```python
# Auto-routing — on first call, the autorouter picks the nearest edge
# that already has the configured model loaded (polargrid-sdk 0.10.0);
# if no edge serves it, routing falls back to the default edge.
config = PolarGridConfig(api_key="pg_...", region=None)

# Pinned — request always goes to the named edge.
config = PolarGridConfig(api_key="pg_...", region="vancouver")
```

The auto-routing path is convenient for development but **pin a region in production** if residency matters. Confirm with PolarGrid whether their auto-routing or failover ever crosses regions before relying on it for compliance.

### Region discovery

`available_regions()` is the curated, offline catalog of all PolarGrid edges (id, name, location); `connected_region()` reports the edge a provider is actually routed to. Both return `PolarGridRegion`, and `location` carries the **Canada / US** split that matters for residency:

```python
from roomkit.providers.polargrid import PolarGridAIProvider

# All edges, filtered to the Canadian ones (Law 25 / PIPEDA).
canadian = [r for r in PolarGridAIProvider.available_regions()
            if (r.location or "").startswith("Canada")]
# → yto-01 Toronto, yul-01 Montreal, yul-02 Montreal 02 (pilot), yvr-02 Vancouver

# Which edge am I actually hitting (esp. under auto-routing)?
here = await provider.connected_region()
print(here.id, here.name, here.location)   # e.g. "yul-01" "Montreal" "Canada East"
```

| Region | Name | Location |
|--------|------|----------|
| `yto-01` | Toronto | Canada Central |
| `yul-01` | Montreal | Canada East |
| `yul-02` | Montreal 02 | Canada East (customer-pilot edge, no longer in the published list) |
| `yvr-02` | Vancouver | Canada West |
| `nyc-01` / `nyc-02` / `was-01` / `mia-01` | New York, Washington DC, Miami | US East |
| `dfw-01` / `dfw-02` / `chi-01` | Dallas, Chicago | US Central |
| `sfo-01` / `sfo-03` / `lax-01` / `sea-01` / `phx-01` | San Francisco, Los Angeles, Seattle, Phoenix | US West |

`available_regions()` mirrors the SDK's own routing table (16 edges in polargrid-sdk 0.10.0). PolarGrid's [regions guide](https://polargrid.mintlify.app/guides/regions) publishes 15 of them: `yul-02` left the public fleet with the `qwen-3.6-35b-a3b` pilot, but the SDK still routes it and the edge still answers, so pinning it is not refused. There is **no live full-region endpoint** (`/v1/status` 404s on edges), so `connected_region()` reports only the routed edge (its `location` is backfilled from the catalog). Every edge answered `/health` on 2026-09-02 except `yto-01`, down at the time.

## Models

Like every RoomKit AI provider, PolarGrid exposes two discovery entry points returning `ModelInfo`:

```python
# Curated, offline catalog — a classmethod, no API key or network.
for m in PolarGridAIProvider.available_models():
    print(m.id, m.display_name, m.capabilities)

# Live query against the connected edge (region-specific).
provider = PolarGridAIProvider(PolarGridConfig(api_key="pg_...", region="yul-01"))
for m in await provider.list_models():
    print(m.id)
```

`available_models()` is the curated snapshot of the **chat** models on the public edges (sourced from PolarGrid's [model pages](https://polargrid.mintlify.app/models) and [model availability guide](https://polargrid.mintlify.app/guides/model-availability), cross-checked against the autorouter on 2026-09-02); `list_models()` returns whatever is actually loaded on the connected edge, **including the STT/TTS models**, and backfills display names, context windows and vision flags from the catalog.

| Model | Type | Availability |
|-------|------|--------------|
| `qwen-3.8-27b` | chat (tools, **thinking**), 256K context, $0.20 / $0.75 per 1M tokens | fleet-wide (every public edge) |
| `qwen-3.6-35b-a3b` | chat (tools, **thinking**, **vision**), 8K served context | **customer pilot**: no public edge; recognised (`supports_vision`, `list_models` backfill) but not advertised |
| `qwen-3.5-27b` | chat | **retired 2026-08-20**: `404 model_not_loaded` everywhere, dropped from the catalog |
| `whisper-large-v3-turbo` | STT | all edges except dfw-02 |
| `cohere-transcribe-03-2026` | STT | all edges except dfw-02 |
| `kokoro-82m` | TTS | all edges except dfw-02 |
| `tada-3b-ml` | TTS | all edges |

The autorouter answers `GET https://autorouter.polargrid.ai/v1/route?model=<id>` with `404` when no edge serves an id, which is how the catalog is checked without a key. The default `qwen-3.8-27b` reasons via `enable_thinking`. See [`examples/list_models.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/list_models.py) for a runnable catalog dump across providers.

## Streaming

PolarGrid streams via OpenAI-shaped chunked SSE. RoomKit's provider exposes both plain text deltas and structured events.

```python
async for delta in provider.generate_stream(context):
    print(delta, end="", flush=True)

async for event in provider.generate_structured_stream(context):
    # StreamThinkingDelta | StreamTextDelta | StreamToolCall | StreamDone
    ...
```

## Thinking / reasoning

qwen surfaces its reasoning **inline** as `<think>...</think>` tags in the message content (the same convention vLLM / Ollama reasoning models use; PolarGrid has no separate `reasoning_content` field). The provider parses those tags out so:

- non-streaming `generate()` returns the reasoning on `AIResponse.thinking` and a clean `AIResponse.content`;
- streaming `generate_structured_stream()` emits the reasoning as `StreamThinkingDelta` (handling tags split across chunks) and the answer as `StreamTextDelta`.

`generate_stream()` (plain text) filters thinking out entirely.

Reasoning is **off by default** on the edge. polargrid-sdk 0.8.5+ exposes an `enable_thinking` request flag, which the provider sets from the `thinking` config:

```python
PolarGridConfig(api_key="pg_...", thinking=True)   # enable_thinking=true  → reasoning on
PolarGridConfig(api_key="pg_...", thinking=False)  # enable_thinking=false → reasoning off
PolarGridConfig(api_key="pg_...")                  # thinking=None         → flag unset (model default)
```

Thinking responses are **larger and slower** (the reasoning counts toward latency and `max_tokens`), so raise `timeout` and `max_tokens` when enabling it. To display reasoning in a CLI, construct the channel with `CLIChannel("cli", show_thinking=True)`.

## Tool / function calling

PolarGrid's chat-completions endpoint supports tool / function calling as of `polargrid-sdk>=0.8.5`. The provider forwards `context.tools` (OpenAI-shaped) and surfaces tool calls back:

- **Non-streaming** — `generate()` returns them on `AIResponse.tool_calls`.
- **Streaming** — `generate_structured_stream()` emits a `StreamToolCall` per call after the text deltas, accumulating the SDK's fragmented `delta.tool_calls`.

PolarGrid sends tool arguments as a JSON string; the provider parses them into a dict for RoomKit (malformed payloads are preserved under a `raw` key). For multi-turn tool loops, assistant tool calls and tool results are rendered back into structured messages (`role="assistant"` with `tool_calls`, `role="tool"` with `tool_call_id`) rather than flattened to text.

```python
from roomkit.providers.ai.base import AITool

context.tools = [
    AITool(
        name="get_weather",
        description="Get current weather for a city.",
        parameters={
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    )
]
```

`tool_choice` is not exposed by `AIContext`, so it is left unset and the backend defaults to `auto`. **Forcing a specific tool is steered, not hard-guaranteed**, on PolarGrid's backend — design tool loops to tolerate the model answering directly instead of calling the tool.

[`examples/polargrid_ai.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/polargrid_ai.py) wires a `web_search` tool end-to-end: it passes `tools=[WebSearchTool()]` to the `AIChannel`, which runs the whole loop (model → `web_search` → grounded answer). Ask it "What is the speed of light?" to see the model search and answer from the result. The tool works key-free (Wikipedia search + summary, which handles the natural-language queries models generate); set `TAVILY_API_KEY` for real web search that also finds niche companies and current info.

## Vision

PolarGrid added multimodal chat in `polargrid-sdk>=0.9.0`: `Message.content` now accepts OpenAI-shaped `image_url` parts. The provider renders an `AIImagePart` (in a user turn or split off an image tool result) as an `image_url` block — the URL may be a remote `https://` URL or a base64 `data:` URI.

`supports_vision` is **model-driven**: it reads the configured model's flag from the curated catalog, pilot models included. Only `qwen-3.6-35b-a3b` actually reads images — verified live on `yul-02` while that edge was public. The default `qwen-3.8-27b` refuses image input outright (`ValidationError`: "does not support image input"), so it — and any unknown model id — is treated as text-only. Vision is the deployed model's capability, not the SDK's, and as of 2026-09-02 no public edge serves a vision-capable model: `qwen-3.6-35b-a3b` is a customer pilot.

With pilot access, pin the vision model and the pilot edge you were given:

```python
PolarGridConfig(api_key="pg_...", model="qwen-3.6-35b-a3b", region="yul-02")
```

[`examples/polargrid_ai.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/polargrid_ai.py) accepts a `/image <path> [question]` command: it embeds the local file as a base64 `data:` URI and sends it as an `image_url` part (defaulting to "Analyse this image." when no question is given). Run it with `POLARGRID_MODEL=qwen-3.6-35b-a3b POLARGRID_REGION=<pilot edge>` for vision.

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
