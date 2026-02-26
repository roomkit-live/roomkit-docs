# AI Thinking / Reasoning

RoomKit provides first-class support for AI thinking (chain-of-thought reasoning). Models like Claude 3.5+, DeepSeek-R1, and QwQ produce internal reasoning before their answer. RoomKit captures this reasoning, preserves it across tool-loop rounds, and exposes it through hooks and ephemeral events.

## Quick start

```python
from roomkit import AIChannel, AnthropicAIProvider, AnthropicConfig

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
from roomkit import AnthropicAIProvider, AnthropicConfig

provider = AnthropicAIProvider(AnthropicConfig(
    api_key="sk-...",
    model="claude-sonnet-4-20250514",
))

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

### Ollama / vLLM (`<think>` tags)

Models served via Ollama or vLLM (DeepSeek-R1, QwQ, etc.) emit reasoning inside `<think>...</think>` tags. The `OpenAIAIProvider` parses these automatically:

- **Streaming**: `_ThinkTagParser` handles tags split across chunk boundaries
- **Non-streaming**: Regex extraction from the complete response
- **History**: `AIThinkingPart` is re-wrapped as `<think>` tags when sent back to the model

```python
from roomkit import create_vllm_provider

provider = create_vllm_provider(
    base_url="http://localhost:11434/v1",
    api_key="ollama",
    model="deepseek-r1:8b",
)

ai = AIChannel("ai", provider=provider, thinking_budget=8192)
```

### Gemini

Gemini does not currently emit thinking content. The `thinking_budget` parameter is accepted but has no effect.

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
from roomkit import AIThinkingPart

part = AIThinkingPart(
    thinking="Let me reason step by step...",
    signature="abc123",  # Optional, used by Anthropic for round-trip
)
```

### StreamThinkingDelta

A streaming event for thinking content:

```python
from roomkit import StreamThinkingDelta

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
from roomkit import AIChannel, MockAIProvider
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
