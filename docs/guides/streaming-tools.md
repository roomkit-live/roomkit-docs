# Streaming with Tools

When an AI provider supports streaming and tools are configured, `AIChannel` uses a **streaming tool loop** that delivers text progressively to downstream channels while executing tool calls between generation rounds. This means WebSocket clients see tokens arrive in real time and Voice/TTS channels can start speaking before the full response is ready -- even when the AI needs to call tools mid-conversation.

## Quick start

```python
from roomkit import AIChannel
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig


class LookupOrderTool:
    @property
    def definition(self) -> dict:
        return {
            "name": "lookup_order",
            "description": "Look up order status by ID",
            "parameters": {
                "type": "object",
                "properties": {"id": {"type": "string"}},
                "required": ["id"],
            },
        }

    async def handler(self, name: str, arguments: dict) -> str:
        return '{"status": "shipped", "eta": "2026-02-20"}'


provider = AnthropicAIProvider(
    AnthropicConfig(api_key="sk-...", model="claude-opus-5")
)
ai = AIChannel(
    "ai-assistant",
    provider=provider,
    system_prompt="You are a helpful support agent.",
    tools=[LookupOrderTool()],
)
```

Pass `Tool` objects via `tools=[]` — definitions and handlers are extracted automatically. When the provider supports structured streaming (Anthropic does natively), the streaming tool loop activates automatically whenever tools are present.

## How it works

```
Round 1: AI generates
    ├── "Let me look that up..."  →  yielded to downstream as text deltas
    └── tool_call: lookup_order(id="ORD-42")
         │
         ▼
    tool handler("lookup_order", {"id": "ORD-42"})  →  '{"status": "shipped"}'
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
from roomkit.providers.ai.base import StreamTextDelta, StreamToolCall, StreamDone, StreamEvent

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
from roomkit.providers.ai.base import AIProvider, AIContext
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

### Max tool calls per round

`max_tool_rounds` bounds how many rounds run; it says nothing about how *wide*
one round may be. A model that degenerates mid-completion spends its whole
output budget emitting tool calls, and every one of them would be honoured —
observed on a 27B local model: 164 calls in a single completion, 154 of them
byte-identical, stopping only at `max_tokens`. That cost 164 executions and
328 room events for one turn.

A round is capped at **32** tool calls. The cap applies before the assistant
message is assembled, not just before execution: truncating the execution
alone would leave a tool call with no matching result in the transcript, which
is a hard error on every OpenAI-compatible provider — trading a loop for a 400
on the next round.

The drop is invisible to the model (the calls it keeps are its own, in its own
order) and loud in the log, which is where an operator diagnoses a looping
model. The cap lives in the shared loop rules, so the streaming and
non-streaming loops enforce it identically.

## Knowing why the loop stopped

The loop ends on rules of its own: the round cap, the wall-clock deadline, a
round truncated at the output cap, a model that answered nothing after its
tools, a cancellation. It knew which one fired and wrote it to the log — but
the stream just ended, so a consumer could not tell a finished answer from a
loop cut mid-work, and had to re-derive the cause by counting tool calls and
reading a clock. That guess is what reports a stopped agent as a model that
returned nothing.

The loop yields a final `LoopEndMarker` on **every** exit, `completed`
included, so the end of the stream is never itself the signal.

Read it at the source, by subclassing `AIChannel` and wrapping
`ChannelOutput.response_stream`:

```python
from roomkit.channels.ai import AIChannel
from roomkit.models.streaming import LoopEndMarker


class ObservingAIChannel(AIChannel):
    async def on_event(self, event, binding, context):
        output = await super().on_event(event, binding, context)
        if output.response_stream is None:
            return output
        return output.model_copy(
            update={"response_stream": self._observe(output.response_stream)}
        )

    async def _observe(self, inner):
        async for delta in inner:
            if isinstance(delta, LoopEndMarker):
                if delta.reason != "completed":
                    logger.warning("agent stopped: %s after %d rounds",
                                   delta.reason, delta.rounds)
                continue          # keep the terminal marker out of the stream
            yield delta
```

!!! note "The marker does not reach downstream channels"
    The framework's inbound streaming path forwards text deltas and the
    tool-call and thinking markers to a channel's `deliver_stream`, but not
    the terminal marker — it would arrive at a renderer as noise. So
    overriding `deliver_stream` on a WebSocket or CLI channel will **not**
    see it. Wrap the AI channel's own `response_stream`, as above, and act on
    the reason there.

| `reason` | The loop stopped because |
|---|---|
| `completed` | The model produced its answer (or the turn ran no tool at all) |
| `max_rounds` | `max_tool_rounds` was reached |
| `timeout` | The wall-clock deadline passed |
| `truncated` | The final round hit the output cap with no text — often reasoning consuming the whole budget |
| `empty_response` | The model answered nothing after its tool rounds, and the bounded retries were spent |
| `cancelled` | The turn was cancelled |

`rounds` is how many tool rounds ran before the stop. The limits each reason
refers to are your own configuration, so they are not repeated on the marker.

The marker is additive by construction rather than by promise: the streaming
protocol is a mixed `str | StreamMarker` and its consumers already dispatch on
the markers they know, so a text-only consumer filtering on
`isinstance(chunk, str)` is unaffected.

**Streaming only.** The non-streaming loop hands back an `AIResponse` the
caller already holds, so its end is not silent in the same way; giving it the
same fact would change that return type.

### Error handling

Exceptions from tool handlers propagate out of the async generator. The framework's streaming delivery infrastructure catches these and logs them, the same as any other streaming error.

## Testing

Use `MockAIProvider` with `ai_responses` and `streaming=True` to test the streaming tool loop:

```python
from roomkit import AIChannel
from roomkit.providers.ai.mock import MockAIProvider
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

ai = AIChannel("ai", provider=provider, tool_handler=handler)  # raw handler for test simplicity
# ... trigger on_event, consume response_stream
```

See [`tests/test_ai_streaming_tool_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/tests/test_ai_streaming_tool_loop.py) for complete test patterns.

## Example

See [`examples/streaming_tools.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/streaming_tools.py) for a runnable demo showing the streaming tool loop with `AIChannel` and `WebSocketChannel`, including an `AIChannel` subclass that reads the `LoopEndMarker` off `response_stream`.
