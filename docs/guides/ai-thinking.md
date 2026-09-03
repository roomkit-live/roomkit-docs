# AI Thinking / Reasoning

RoomKit provides first-class support for AI thinking (chain-of-thought reasoning). Models like Claude 3.5+, DeepSeek-R1, and QwQ produce internal reasoning before their answer. RoomKit captures this reasoning, preserves it across tool-loop rounds, and exposes it through hooks and ephemeral events.

## Quick start

```python
from roomkit import AIChannel
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig

provider = AnthropicAIProvider(
    AnthropicConfig(api_key="sk-...", model="claude-opus-5")
)
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

Three settings steer reasoning, and all three resolve through the same chain:

| Parameter | Type | Description |
|-----------|------|-------------|
| `thinking_budget` | `int \| None` | Token budget for reasoning |
| `enable_thinking` | `bool \| None` | Turn the reasoning block on or off, for providers that expose the switch |
| `reasoning_effort` | `str \| None` | Reasoning verbosity, for providers that grade it — the accepted values are the provider's own |

`None` everywhere means "not set at this tier", so an unset knob defers
outward rather than overriding with a default.

### The resolution chain

Each setting is resolved fresh at the start of every turn, from the most
specific source that has an opinion:

```
1. Binding metadata          → per-room operator intent, always wins
2. config_provider result    → resolved by your callback, every turn
3. AIChannel constructor     → the channel default
4. Provider config           → the provider's own setting
5. (nothing set)             → the model's own default
```

### Channel default

Set the default when creating the channel:

```python
ai = AIChannel(
    "ai-thinker",
    provider=provider,
    thinking_budget=8192,
    enable_thinking=True,
    reasoning_effort="low",
)
```

### Per-room override

Override for specific rooms via binding metadata:

```python
await kit.attach_channel("math-room", "ai-thinker",
    category=ChannelCategory.INTELLIGENCE,
    metadata={
        "system_prompt": "You are a math tutor. Show your work.",
        "thinking_budget": 16384,   # More budget for complex reasoning
        "reasoning_effort": "high",
    },
)
```

### Per-turn override (`config_provider`)

A thinking model costs two to three times the tokens and the latency of a
direct answer, and that trade is not the same in an agent's tool loop — where
the model is mostly shaping results it already has — as in a chat turn where
the reasoning *is* the value. `config_provider` lets you decide per turn
rather than standing up a second channel and a second provider to say so.

The callback runs at the start of every generation and returns an
`AIChannelTurnConfig`; `None` fields fall through to the tiers below:

```python
from roomkit import AIChannelTurnConfig

async def per_turn(binding, context) -> AIChannelTurnConfig | None:
    # Cheap, direct answers while the agent is working through its tools;
    # full reasoning once it is composing the answer for a person.
    if context.room.metadata.get("mode") == "agent":
        return AIChannelTurnConfig(enable_thinking=False)
    return AIChannelTurnConfig(enable_thinking=True, reasoning_effort="high")

ai = AIChannel("ai-thinker", provider=provider, config_provider=per_turn)
```

`AIChannelTurnConfig` carries `system_prompt`, `tools`, `temperature`,
`max_tokens`, `thinking_budget`, `enable_thinking` and `reasoning_effort`.
Because it is resolved every turn, it is the right place for config that
changes underneath you — admin edits, per-user gating, feature flags — where
snapshotting into the channel or the binding at attach time would go stale.

### When reasoning eats the output budget

Reasoning and the answer compete for the same `max_tokens`. A model that
spends its whole budget inside the thinking block returns empty `content`
with a truncation finish reason — not silence, but a cap that was too small
for both halves. RoomKit recognises that case and does **not** waste a retry
re-prompting under the same cap; it logs the real cause instead. If you see
it, raise `max_tokens`, lower `reasoning_effort`, or set
`enable_thinking=False` for that turn.

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

- **Streaming**: `ThinkTagParser` handles tags split across chunk boundaries
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
`headers` adds proxy/non-Bearer headers.

```python
provider = create_vllm_provider(VLLMConfig(
    base_url="http://gpu-server:8000/v1",
    model="meta-llama/Llama-3.1-8B-Instruct",
    api_key="token-abc123",                  # vllm serve --api-key token-abc123
    headers={"X-Proxy-Region": "eu"},        # optional reverse-proxy headers
    top_k=40,                                # typed, no extra_body needed
    repetition_penalty=1.05,
    extra_body={
        "guided_choice": ["yes", "no"],      # constrain output to a choice set
    },
))
```

#### vLLM sampling knobs

`top_p`, `top_k`, `min_p`, `presence_penalty` and `repetition_penalty` are
declared `VLLMConfig` fields, reaching parity with `OllamaConfig`. They were
always reachable through `extra_body`, but that left the caller to know which
are OpenAI fields and which are vLLM extensions; the config now routes all
five through the request body itself.

Each defaults to `None`, meaning "the server decides" — which is not the same
as sending the documented default, and is the only honest answer for a model
RoomKit cannot see. An explicit `0` survives: `min_p=0.0` and
`presence_penalty=0.0` are values, not absences.

This is what makes a vendor's published sampling profile expressible. Qwen3
asks for `presence_penalty=1.5` in non-thinking mode, and the failure that
setting addresses is degenerate repetition:

```python
provider = create_vllm_provider(VLLMConfig(
    model="Qwen/Qwen3-8B",
    enable_thinking=False,
    presence_penalty=1.5,   # Qwen3's own guidance for non-thinking mode
    top_p=0.8,
    top_k=20,
    min_p=0.0,
))
```

#### vLLM reasoning knobs

vLLM renders the model's chat template **server-side**, so reasoning is
steered through `chat_template_kwargs` rather than a sampling parameter — the
top-level `reasoning_effort` an OpenAI-compatible client sends is not read by
a locally rendered template. `enable_thinking` and `reasoning_effort` map onto
those template kwargs, so a thinking model can be told to answer directly
without hand-writing `extra_body`:

```python
provider = create_vllm_provider(VLLMConfig(
    model="Qwen/Qwen3-8B",
    enable_thinking=False,     # → chat_template_kwargs
))
```

Both default to `None`, leaving the model's own default untouched. That
default matters: current Qwen builds think at their **most verbose effort**
unless told otherwise, and in a tool loop that reasoning competes with the
answer for the same `max_tokens`.

An explicit `extra_body["chat_template_kwargs"]` entry still wins, so the
escape hatch keeps working for templates this config does not model. A
per-turn setting resolves *over* the configured one by merging rather than
replacing, so a turn that switches `enable_thinking` cannot silently drop a
configured `reasoning_effort` it says nothing about.

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

A round can bracket **more than one** reasoning phase. A model that reasons, answers, then reasons again — the shape Anthropic's interleaved thinking produces — opens and closes a window per switch, so subscribers see several `THINKING_START` / `THINKING_END` pairs for a single round. Each `THINKING_END` carries **its own block only**, never the blocks the earlier ones already delivered: a client appends what it receives and never has to de-duplicate. The reasoning kept in conversation history stays whole, all blocks of the round concatenated.

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

### AIContext fields

Resolved per turn by the chain above and handed to the provider:

| Field | Type | Description |
|-------|------|-------------|
| `thinking_budget` | `int \| None` | Token budget for reasoning |
| `enable_thinking` | `bool \| None` | Reasoning block on/off; `None` defers to the provider config, then to the model |
| `reasoning_effort` | `str \| None` | Reasoning verbosity; accepted values are the provider's own |
| `max_tokens` | `int \| None` | Output cap for this turn; `None` defers to the provider's configured `max_tokens` |

`max_tokens` defaults to `None` rather than a number on purpose. A non-`None`
default here would shadow every provider config — `context.max_tokens or
self._config.max_tokens` would never reach the second term, and a configured
cap would be unreachable.

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

See [`examples/ai_turn_config.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/ai_turn_config.py) for the per-turn `config_provider` chain, printing what each tier resolved to on the way to the provider.
