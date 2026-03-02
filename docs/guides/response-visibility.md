# Response Visibility

`response_visibility` controls where an AI channel's response gets delivered. While `visibility` controls who **receives** the inbound event, `response_visibility` controls where the AI's **reply** goes -- using the same value vocabulary (`"all"`, `"none"`, `"transport"`, `"intelligence"`, channel ID, or comma-separated IDs).

## Quick start

```python
from roomkit import HookTrigger, HookResult, RoomKit

kit = RoomKit()

@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def route_response(event, context):
    if event.source.channel_id == "text-input":
        return HookResult(
            action="modify",
            event=event.model_copy(update={
                "visibility": "ai",              # only AI sees the message
                "response_visibility": "ws-ui",   # AI reply goes to WebSocket only
            }),
        )
    return HookResult(action="allow")
```

When the AI produces a response, it will be delivered only to the `ws-ui` channel -- voice and other transports are skipped.

## How it works

```
User types in text UI
    │
    ▼
RoomEvent(visibility="ai", response_visibility="ws-ui")
    │
    ├── broadcast to AI channel (visibility="ai" allows it)
    │       │
    │       ▼
    │   AI generates response
    │       │
    │       ├── Streaming path: _handle_streaming_response()
    │       │     • filters streaming targets by response_visibility
    │       │     • stamps visibility="ws-ui" on stored event
    │       │
    │       └── Non-streaming path: reentry drain loop
    │             • stamps visibility="ws-ui" on reentry events
    │
    └── broadcast AI response with visibility="ws-ui"
            │
            ├── ws-ui: _check_visibility → allowed ✓
            └── voice:  _check_visibility → blocked ✗
```

The mechanism works by transferring `response_visibility` from the trigger event to the AI's response event as `visibility`. Once stamped, the existing `_check_visibility()` in the broadcast pipeline handles all the filtering -- no special cases needed downstream.

## Value vocabulary

`response_visibility` accepts the same values as `visibility`:

| Value | Effect on AI response delivery |
|-------|-------------------------------|
| `None` (default) | No restriction -- delivers to all channels (`"all"`) |
| `"all"` | Delivers to all channels |
| `"none"` | No delivery -- response is stored only |
| `"transport"` | Only transport channels (SMS, WebSocket, Voice, etc.) |
| `"intelligence"` | Only intelligence channels (other AI channels) |
| `"ws-ui"` | Only the channel with ID `ws-ui` |
| `"ws-ui,sms-out"` | Only channels `ws-ui` and `sms-out` |

## Setting response_visibility

### Via BEFORE_BROADCAST hook

The most flexible approach. Inspect the event and decide where the response should go based on metadata, source channel, or any runtime condition:

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def route_response(event, context):
    # Route based on metadata set by the transport layer
    reply_to = event.metadata.get("reply_to_channel")
    if reply_to:
        return HookResult(
            action="modify",
            event=event.model_copy(update={
                "response_visibility": reply_to,
            }),
        )
    return HookResult(action="allow")
```

### Via send_event

When injecting events programmatically:

```python
await kit.send_event(
    room_id=room_id,
    channel_id="voice",
    content=TextContent(body=user_text),
    visibility="ai",
    response_visibility="ws-ui",
)
```

### Via custom channel

A channel can stamp `response_visibility` in its `handle_inbound`:

```python
class TextInputChannel(Channel):
    async def handle_inbound(self, message, context):
        return RoomEvent(
            room_id=context.room.id,
            source=EventSource(
                channel_id=self.channel_id,
                channel_type=self.channel_type,
            ),
            content=message.content,
            response_visibility="ws-ui",
        )
```

## Streaming and non-streaming paths

`response_visibility` works with both AI response delivery modes:

- **Streaming** (e.g. Anthropic with `supports_streaming=True`): the streaming target selection in `_handle_streaming_response()` filters channels before piping text deltas. The stored response event gets `visibility` stamped so the reentry broadcast also respects it.

- **Non-streaming** (e.g. `MockAIProvider`): the reentry drain loop in `_process_locked()` stamps `visibility` on the AI's response events before re-broadcasting them.

In both cases, the stored event in the conversation history carries the visibility, which means `get_timeline()` with `visibility_filter` also works correctly.

## Testing

Use `_SourceChannel` with `response_visibility` or a BEFORE_BROADCAST hook to test delivery scope:

```python
from roomkit import MockAIProvider, RoomKit, HookTrigger
from roomkit.channels.ai import AIChannel

kit = RoomKit()
# ... register channels, create room, attach ...

@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def stamp(event, context):
    if event.source.channel_id == "src":
        return HookResult(
            action="modify",
            event=event.model_copy(update={"response_visibility": "t1"}),
        )
    return HookResult(action="allow")

await kit.process_inbound(
    InboundMessage(channel_id="src", sender_id="u1", content=TextContent(body="Hi"))
)

# t1 receives the AI response, t2 does not
assert len(t1.ai_delivered) == 1
assert len(t2.ai_delivered) == 0
```

See [`tests/test_response_visibility.py`](https://github.com/roomkit-live/roomkit/blob/main/tests/test_response_visibility.py) for the full test suite covering streaming, non-streaming, `"none"`, comma-separated IDs, and stored event verification.

## Example

See [`examples/response_visibility.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/response_visibility.py) for a runnable demo showing a hybrid voice+text setup where the AI response is routed to a specific WebSocket channel.
