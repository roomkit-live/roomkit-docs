# Streaming with Tools

When an AI provider supports streaming and tools are configured, `AIChannel` uses a **streaming tool loop** that delivers text progressively to downstream channels while executing tool calls between generation rounds. This means WebSocket clients see tokens arrive in real time and Voice/TTS channels can start speaking before the full response is ready -- even when the AI needs to call tools mid-conversation.

## Quick start

```python
from roomkit import AIChannel, AnthropicAIProvider, AnthropicConfig

async def tool_handler(name: str, arguments: dict) -> str:
    if name == "lookup_order":
        return '{"status": "shipped", "eta": "2026-02-20"}'
    return '{"error": "Unknown tool"}'

provider = AnthropicAIProvider(AnthropicConfig(api_key="sk-..."))
ai = AIChannel(
    "ai-assistant",
    provider=provider,
    system_prompt="You are a helpful support agent.",
    tool_handler=tool_handler,
)
```

That's it. When the provider supports structured streaming (Anthropic does natively), the streaming tool loop activates automatically whenever tools are present.

## How it works

```
Round 1: AI generates
    ├── "Let me look that up..."  →  yielded to downstream as text deltas
    └── tool_call: lookup_order(id="ORD-42")
         │
         ▼
    tool_handler("lookup_order", {"id": "ORD-42"})  →  '{"status": "shipped"}'
         │
         ▼
Round 2: AI generates (with tool result in context)
    └── "Your order ORD-42 has shipped!"  →  yielded to downstream as text deltas
         │
         ▼
    No more tool calls → stream ends
```

The framework's existing streaming delivery infrastructure (`deliver_stream`, `stream_start`/`stream_chunk`/`stream_end` for WebSocket, sentence-splitting for Voice/TTS) requires **zero changes**. The streaming tool loop yields `str` text deltas just like `generate_stream()` does, so all downstream channels work transparently.

## Structured stream events

The streaming tool loop is built on three event types emitted by `AIProvider.generate_structured_stream()`:

| Event | Fields | Description |
|-------|--------|-------------|
| `StreamTextDelta` | `text` | A chunk of generated text |
| `StreamToolCall` | `id`, `name`, `arguments` | A complete tool call extracted after streaming |
| `StreamDone` | `finish_reason`, `usage`, `metadata` | Signals the end of one generation round |

These are Pydantic models exported from `roomkit`:

```python
from roomkit import StreamTextDelta, StreamToolCall, StreamDone, StreamEvent

# StreamEvent is the union type
event: StreamEvent = StreamTextDelta(text="Hello")
```

## Provider support

### Anthropic (native)

`AnthropicAIProvider` has native structured streaming support. It uses the Anthropic SDK's `messages.stream()` context manager, yielding `StreamTextDelta` events from the text stream in real time. After the stream completes, it calls `get_final_message()` to extract any tool calls and usage data.

Because `generate()` now delegates to `generate_structured_stream()` internally, it inherently uses the streaming API. This also avoids the Anthropic SDK's timeout limitation that rejects non-streaming requests with high `max_tokens` values.

### Other providers (fallback)

Providers that don't override `generate_structured_stream()` use the default fallback, which wraps `generate()`:

1. Calls `generate()` to get a complete `AIResponse`
2. Yields `StreamTextDelta` with the full response text
3. Yields `StreamToolCall` for each tool call in the response
4. Yields `StreamDone` with finish reason and usage

This means every provider works with the streaming tool loop without changes, but text delivery is not truly incremental -- the full response arrives as a single delta. To get progressive delivery, providers should override `generate_structured_stream()` and set `supports_structured_streaming = True`.

### Implementing for a custom provider

```python
from roomkit import AIProvider, AIContext
from roomkit.providers.ai.base import (
    StreamTextDelta, StreamToolCall, StreamDone, StreamEvent,
)
from collections.abc import AsyncIterator

class MyProvider(AIProvider):
    @property
    def supports_structured_streaming(self) -> bool:
        return True

    @property
    def supports_streaming(self) -> bool:
        return True

    async def generate_structured_stream(
        self, context: AIContext
    ) -> AsyncIterator[StreamEvent]:
        # Stream text deltas as they arrive
        async for chunk in self._my_streaming_api(context):
            if chunk.type == "text":
                yield StreamTextDelta(text=chunk.text)

        # After streaming, yield any tool calls
        for tool_call in self._pending_tool_calls:
            yield StreamToolCall(
                id=tool_call.id,
                name=tool_call.name,
                arguments=tool_call.arguments,
            )

        yield StreamDone(
            finish_reason="stop",
            usage={"input_tokens": 100, "output_tokens": 50},
        )
```

## Streaming tool loop details

### Routing logic

`AIChannel.on_event()` routes based on provider capabilities:

| `supports_streaming` | `supports_structured_streaming` | Has tools | Path |
|---|---|---|---|
| `True` | any | No | `_start_streaming_response` (plain `generate_stream`) |
| any | `True` | No | `_start_streaming_response` (plain `generate_stream`) |
| `True` | any | Yes | `_start_streaming_tool_response` (streaming tool loop) |
| any | `True` | Yes | `_start_streaming_tool_response` (streaming tool loop) |
| `False` | `False` | any | `_generate_response` (non-streaming with tool loop) |

### Max rounds

The `max_tool_rounds` parameter (default 10) controls how many times tools can be executed. The loop runs at most `max_tool_rounds + 1` generations:

- Generations 0 through `max_tool_rounds - 1`: if tool calls are returned, tools are executed and the loop continues
- Generation `max_tool_rounds`: final generation only -- tool calls are **not** executed (since no generation would follow to use the results)

This matches the non-streaming tool loop semantics and prevents side-effecting tools from executing when their results would be discarded.

### Error handling

Exceptions from `tool_handler` propagate out of the async generator. The framework's streaming delivery infrastructure catches these and logs them, the same as any other streaming error.

## Testing

Use `MockAIProvider` with `ai_responses` and `streaming=True` to test the streaming tool loop:

```python
from roomkit import AIChannel, MockAIProvider
from roomkit.providers.ai.base import AIResponse, AIToolCall

responses = [
    # Round 1: tool call
    AIResponse(
        content="Looking up your order...",
        finish_reason="tool_calls",
        usage={"prompt_tokens": 10, "completion_tokens": 5},
        tool_calls=[
            AIToolCall(id="tc1", name="lookup_order", arguments={"id": "ORD-42"}),
        ],
    ),
    # Round 2: final text
    AIResponse(
        content="Your order has shipped!",
        finish_reason="stop",
        usage={"prompt_tokens": 20, "completion_tokens": 10},
    ),
]

provider = MockAIProvider(ai_responses=responses, streaming=True)

async def handler(name, args):
    return '{"status": "shipped"}'

ai = AIChannel("ai", provider=provider, tool_handler=handler)
# ... trigger on_event, consume response_stream
```

See [`tests/test_ai_streaming_tool_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/tests/test_ai_streaming_tool_loop.py) for complete test patterns.

## Example

See [`examples/streaming_tools.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/streaming_tools.py) for a runnable demo showing the streaming tool loop with `AIChannel` and `WebSocketChannel`.
