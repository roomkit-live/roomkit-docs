# AI Thinking / Reasoning

RoomKit provides first-class support for AI thinking (chain-of-thought reasoning). Models like Claude 3.5+, DeepSeek-R1, and QwQ produce internal reasoning before their answer. RoomKit captures this reasoning, preserves it across tool-loop rounds, and exposes it through hooks and ephemeral events.

## Quick start

```python
from roomkit import AIChannel
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig

provider = AnthropicAIProvider(AnthropicConfig(api_key="sk-..."))
ai = AIChannel(
    "ai-thinker",
    provider=provider,
    system_prompt="Think step by step before answering.",
    thinking_budget=8192,  # Token budget for reasoning
)
```

That's it. When the provider supports thinking, the reasoning is automatically captured and preserved in conversation history.

## How it works

```
User message arrives
    │
    ▼
AIChannel builds AIContext (with thinking_budget)
    │
    ▼
Provider generates response
    ├── Thinking: "Let me reason step by step..."  →  THINKING_START ephemeral event
    │                                               →  ON_AI_THINKING hook
    │                                               →  THINKING_END ephemeral event
    └── Answer: "The answer is 42."                →  Broadcast as RoomEvent
    │
    ▼
AIThinkingPart preserved in conversation history
    │
    ▼
Next generation sees prior reasoning (required by Anthropic, useful for all)
```

## Configuration

| Parameter | Location | Description |
|-----------|----------|-------------|
| `thinking_budget` | `AIChannel()` constructor | Default token budget for reasoning |
| `thinking_budget` | Binding metadata | Per-room override |

### Default thinking budget

Set the default budget when creating the channel:

```python
ai = AIChannel(
    "ai-thinker",
    provider=provider,
    thinking_budget=8192,
)
```

### Per-room override

Override the budget for specific rooms via binding metadata:

```python
await kit.attach_channel("math-room", "ai-thinker",
    category=ChannelCategory.INTELLIGENCE,
    metadata={
        "system_prompt": "You are a math tutor. Show your work.",
        "thinking_budget": 16384,  # More budget for complex reasoning
    },
)
```

## Provider support

### Anthropic (native extended thinking)

`AnthropicAIProvider` uses the native extended thinking API. When `thinking_budget` is set:

- The API receives `thinking: {type: "enabled", budget_tokens: N}`
- Temperature is automatically set to 1 (required by the API)
- Thinking blocks include a `signature` for round-trip fidelity
- `AIThinkingPart` is preserved verbatim in conversation history (Anthropic requires this)

```python
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig

provider = AnthropicAIProvider(AnthropicConfig(
    api_key="sk-...",
    model="claude-opus-5",
))

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

### Ollama / vLLM (`<think>` tags)

Models served via Ollama or vLLM (DeepSeek-R1, QwQ, etc.) emit reasoning inside `<think>...</think>` tags. The `OpenAIAIProvider` parses these automatically:

- **Streaming**: `_ThinkTagParser` handles tags split across chunk boundaries
- **Non-streaming**: Regex extraction from the complete response
- **History**: `AIThinkingPart` is re-wrapped as `<think>` tags when sent back to the model

```python
from roomkit.providers.vllm import VLLMConfig, create_vllm_provider

provider = create_vllm_provider(VLLMConfig(
    base_url="http://localhost:11434/v1",
    api_key="ollama",
    model="deepseek-r1:8b",
))

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

#### vLLM authentication & native params

`create_vllm_provider` wraps `OpenAIAIProvider` — vLLM's online server *is*
the OpenAI-compatible API, so this is the canonical integration. Set
`api_key` to match `vllm serve --api-key` (sent as `Authorization: Bearer`).
`headers` adds proxy/non-Bearer headers; `extra_body` forwards vLLM-specific
request fields the OpenAI schema omits — guided decoding and extra sampling.

```python
provider = create_vllm_provider(VLLMConfig(
    base_url="http://gpu-server:8000/v1",
    model="meta-llama/Llama-3.1-8B-Instruct",
    api_key="token-abc123",                  # vllm serve --api-key token-abc123
    headers={"X-Proxy-Region": "eu"},        # optional reverse-proxy headers
    extra_body={                             # vLLM-native params
        "top_k": 40,
        "repetition_penalty": 1.05,
        "guided_choice": ["yes", "no"],      # constrain output to a choice set
    },
))
```

#### Native Ollama provider & authentication

For Ollama specifically, prefer the native `OllamaAIProvider`. It calls
`/api/chat` directly, so the `think` parameter and the streamed `thinking`
field work without `<think>` tag parsing.

To reach a protected endpoint — Ollama Cloud/Turbo, or a self-hosted server
behind a reverse proxy — set `api_key`; it is sent as
`Authorization: Bearer <key>`. Use `headers` for extra proxy headers or a
non-Bearer scheme (`api_key` wins over an `Authorization` entry in `headers`).
When `api_key` is `None`, the SDK still falls back to the `OLLAMA_API_KEY`
environment variable.

```python
from roomkit.providers.ollama import OllamaAIProvider, OllamaConfig

provider = OllamaAIProvider(OllamaConfig(
    host="https://ollama.example.com",
    model="deepseek-r1:8b",
    api_key="sk-...",                 # → Authorization: Bearer sk-...
    headers={"X-Proxy-Region": "eu"},  # optional extra headers
    think="high",
))

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

`OllamaConfig` also exposes per-config sampling options, mapped to Ollama's
`options`: `temperature` (default `0.7`), `max_tokens` (→ `num_predict`),
`num_ctx`, `top_p`, `top_k`, and `min_p`. Each defaults to `None` (the model's
own default) except `temperature`.

```python
provider = OllamaAIProvider(OllamaConfig(
    model="llama3.2",
    temperature=0.2,
    num_ctx=8192,
    top_p=0.9,
    keep_alive=-1,    # keep the model loaded indefinitely
))
```

`keep_alive` controls how long the model stays resident after a request. Ollama
reads a **string** `keep_alive` as a Go duration (e.g. `"5m"`), so a unit-less
value must be a number, not a numeric string: pass `keep_alive=-1` (load
forever) or `keep_alive=0` (unload immediately) as an `int`. A unit-less string
like `"-1"` is coerced to `int` automatically so Ollama doesn't reject it as a
malformed duration.

### Gemini (thought summaries + thought signatures)

`GeminiAIProvider` streams thought summaries: the parts Gemini flags
`thought=True` surface as `StreamThinkingDelta`, everything else as
`StreamTextDelta`. Two knobs reach the same `ThinkingConfig`, and
`thinking_level` wins when both are set:

- `GeminiConfig(thinking_level=...)` — `minimal`, `low`, `medium`, `high`, for
  Gemini 3.x models
- `thinking_budget` (per turn, from the channel) — a token budget, for Gemini 2.5

```python
from roomkit.providers.gemini import GeminiAIProvider, GeminiConfig

provider = GeminiAIProvider(GeminiConfig(
    api_key="...",
    model="gemini-3.6-flash",
    thinking_level="high",
))
```

When the model reasons and calls tools, each function call it returns carries a
**thought signature**, and Gemini 3 rejects a later turn whose history replays a
function call without one. RoomKit handles the round trip for you: the signature
is kept in `AIToolCallPart.metadata["thought_signature"]` and replayed on the
matching call.

A round of parallel calls is the case to know about — Gemini signs one call of
the group, not all of them, so the provider lends that signature to the round's
other calls when it rebuilds the history. Nothing to configure. If a whole round
comes back unsigned there is nothing to lend, and the provider logs a warning
naming the calls before Gemini rejects the next turn.

RoomKit also refuses, before the request leaves, a history that ends on a model
turn — Gemini answers a user turn and would reply `400 "Requests ending with a
model turn are not supported."` The `ProviderError` names the condition instead,
because the cause is upstream: a turn generated with nothing new to answer,
typically concurrent turns on one room each rebuilding a history that ends on
another's reply.

## Streaming

During streaming generation, thinking content arrives as `StreamThinkingDelta` events before `StreamTextDelta` events:

```python
from roomkit.providers.ai.base import (
    StreamThinkingDelta,
    StreamTextDelta,
    StreamToolCall,
    StreamDone,
)

async for event in provider.generate_structured_stream(context):
    if isinstance(event, StreamThinkingDelta):
        print(f"Thinking: {event.thinking}")
    elif isinstance(event, StreamTextDelta):
        print(f"Text: {event.text}")
```

The `AIChannel` handles this automatically — thinking deltas trigger ephemeral events, and text deltas are delivered to downstream channels.

## Hooks and ephemeral events

### ON_AI_THINKING hook

Fires when the AI produces thinking content. Use it for logging, observability, or cost tracking:

```python
from roomkit import HookTrigger

@kit.hook(HookTrigger.ON_AI_THINKING)
async def log_thinking(event, ctx):
    thinking = ctx.get("thinking", "")
    print(f"AI reasoning ({len(thinking)} chars): {thinking[:100]}...")
```

### Ephemeral events

Two ephemeral events bracket the thinking phase:

| Event | When |
|-------|------|
| `THINKING_START` | AI begins reasoning |
| `THINKING_END` | AI finishes reasoning (thinking text in payload) |

These are published via the `RealtimeBackend` and do not persist in the conversation store. Use them for real-time UI indicators (e.g., "AI is thinking...").

## Tool loop integration

Thinking is preserved across tool-loop rounds. When the AI calls a tool and then continues generating, the thinking from each round is kept in the conversation history:

```
Round 1: AI thinks → calls tool
    ├── AIThinkingPart(thinking="I need to look up...")
    └── AIToolCallPart(name="search", ...)

Tool executes → result appended

Round 2: AI thinks → generates answer
    ├── AIThinkingPart(thinking="Based on the results...")
    └── AITextPart(text="Here's what I found...")
```

This ensures the model has full context of its prior reasoning when generating follow-up responses.

## Data model

### AIThinkingPart

Represents a thinking block in conversation history:

```python
from roomkit.providers.ai.base import AIThinkingPart

part = AIThinkingPart(
    thinking="Let me reason step by step...",
    signature="abc123",  # Optional, used by Anthropic for round-trip
)
```

### StreamThinkingDelta

A streaming event for thinking content:

```python
from roomkit.providers.ai.base import StreamThinkingDelta

delta = StreamThinkingDelta(thinking="Step 1: Consider...")
```

### AIResponse fields

| Field | Type | Description |
|-------|------|-------------|
| `thinking` | `str \| None` | Accumulated thinking text |
| `thinking_signature` | `str \| None` | Provider-specific signature (Anthropic) |

### AIContext field

| Field | Type | Description |
|-------|------|-------------|
| `thinking_budget` | `int \| None` | Token budget for reasoning |

## Testing

Use `MockAIProvider` with `AIResponse` that includes thinking content:

```python
from roomkit import AIChannel
from roomkit.providers.ai.mock import MockAIProvider
from roomkit.providers.ai.base import AIResponse

provider = MockAIProvider(
    ai_responses=[
        AIResponse(
            content="The answer is 42.",
            thinking="Let me reason about this...",
            finish_reason="stop",
            usage={"prompt_tokens": 20, "completion_tokens": 15},
        ),
    ],
    streaming=True,
)

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

When `streaming=True`, `MockAIProvider.generate_structured_stream()` yields `StreamThinkingDelta` before `StreamTextDelta`, matching the real provider behavior.

## Example

See [`examples/ai_thinking.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/ai_thinking.py) for a runnable demo showing thinking with `AIChannel` and per-room configuration.
