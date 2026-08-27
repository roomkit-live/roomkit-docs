# Realtime Features

RoomKit supports two event categories: **persistent events** stored in the `ConversationStore` (messages, edits, deletions) and **ephemeral events** delivered in real time through the `RealtimeBackend` (typing indicators, presence, reactions, read receipts). This guide covers both.

## Event Categories

| Category | Stored | Delivered via | Examples |
|----------|--------|---------------|----------|
| **Persistent** | Yes | `process_inbound()` pipeline + broadcast | Messages, edits, deletions |
| **Ephemeral** | No | `RealtimeBackend` pub/sub | Typing, presence, reactions, tool calls |

---

## Typing Indicators

```python
from __future__ import annotations

from roomkit import RoomKit

kit = RoomKit()

# User starts typing
await kit.publish_typing("room-1", "alice", is_typing=True)

# User stops typing
await kit.publish_typing("room-1", "alice", is_typing=False)

# With additional metadata
await kit.publish_typing("room-1", "alice", is_typing=True, data={"name": "Alice"})
```

## Presence Tracking

Three standard statuses map to dedicated event types:

```python
await kit.publish_presence("room-1", "alice", "online")
await kit.publish_presence("room-1", "alice", "away")
await kit.publish_presence("room-1", "alice", "offline")

# Custom status → EphemeralEventType.CUSTOM with data={"status": "in_meeting"}
await kit.publish_presence("room-1", "alice", "in_meeting")
```

## Reactions

Add or remove emoji reactions on messages:

```python
# Add a reaction
await kit.publish_reaction("room-1", "alice", target_event_id, emoji="thumbsup")

# Remove a reaction
await kit.publish_reaction("room-1", "alice", target_event_id, emoji="thumbsup", action="remove")
```

## Read Receipts

Two complementary mechanisms — ephemeral for UI, persistent for state:

```python
# Ephemeral: "seen" indicator (not stored)
await kit.publish_read_receipt("room-1", "alice", event_id)

# Persistent: durable read state (stored in ConversationStore)
await kit.mark_read("room-1", "ws-alice", event_id)
await kit.mark_all_read("room-1", "ws-alice")
```

!!! note
    `publish_read_receipt` takes a `user_id`, while `mark_read` takes a `channel_id`. Ephemeral receipts are user-facing; persistent state is per-channel.

---

## Subscribing to Events

Subscribe to all ephemeral events in a room:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.realtime import EphemeralEvent, EphemeralEventType

kit = RoomKit()


async def on_event(event: EphemeralEvent) -> None:
    if event.type == EphemeralEventType.TYPING_START:
        print(f"{event.user_id} is typing...")
    elif event.type == EphemeralEventType.PRESENCE_ONLINE:
        print(f"{event.user_id} came online")
    elif event.type == EphemeralEventType.REACTION:
        print(f"{event.user_id} reacted with {event.data['emoji']}")
    elif event.type == EphemeralEventType.TOOL_CALL_START:
        tools = event.data["tool_calls"]
        print(f"AI calling: {[t['name'] for t in tools]}")


sub_id = await kit.subscribe_room("room-1", on_event)

# Later:
await kit.unsubscribe_room(sub_id)
```

## EphemeralEventType

All 14 ephemeral event types:

| Type | Published by |
|------|-------------|
| `TYPING_START` | `publish_typing(is_typing=True)` |
| `TYPING_STOP` | `publish_typing(is_typing=False)` |
| `PRESENCE_ONLINE` | `publish_presence(status="online")` |
| `PRESENCE_AWAY` | `publish_presence(status="away")` |
| `PRESENCE_OFFLINE` | `publish_presence(status="offline")` |
| `READ_RECEIPT` | `publish_read_receipt()` |
| `TOOL_CALL_DELTA` | Automatic from AIChannel |
| `TOOL_CALL_START` | Automatic from AIChannel |
| `TOOL_CALL_END` | Automatic from AIChannel |
| `THINKING_START` | Automatic from AIChannel |
| `THINKING_DELTA` | Automatic from AIChannel |
| `THINKING_END` | Automatic from AIChannel |
| `REACTION` | `publish_reaction()` |
| `CUSTOM` | Direct publish |

---

## Message Editing

Edits are **persistent events** — stored and broadcast to all channels:

```python
from __future__ import annotations

from roomkit import EventType, InboundMessage, RoomKit, TextContent
from roomkit.models.events import EditContent

kit = RoomKit()

await kit.process_inbound(InboundMessage(
    channel_id="ws-alice",
    sender_id="alice",
    event_type=EventType.EDIT,
    content=EditContent(
        target_event_id=original_msg_id,
        new_content=TextContent(body="Fixed typo!"),
    ),
))
```

The `new_content` field accepts any `EventContent` type — you can change a text message to rich content or vice versa.

## Message Deletion

Deletions are also **persistent events**:

```python
from __future__ import annotations

from roomkit import EventType, InboundMessage, RoomKit
from roomkit.models.enums import DeleteType
from roomkit.models.events import DeleteContent

kit = RoomKit()

await kit.process_inbound(InboundMessage(
    channel_id="ws-alice",
    sender_id="alice",
    event_type=EventType.DELETE,
    content=DeleteContent(
        target_event_id=msg_id,
        delete_type=DeleteType.SENDER,
        reason="Changed my mind",
    ),
))
```

| DeleteType | Meaning |
|-----------|---------|
| `SENDER` | Original sender deletes their message |
| `SYSTEM` | System-initiated (e.g., content policy) |
| `ADMIN` | Admin/moderator removal |

---

## Tool Call Events

AIChannel automatically publishes `TOOL_CALL_DELTA`, `TOOL_CALL_START` and `TOOL_CALL_END` — subscribe to observe:

```python
async def on_tool(event: EphemeralEvent) -> None:
    if event.type == EphemeralEventType.TOOL_CALL_DELTA:
        call = event.data["tool_calls"][0]
        print(f"Composing {call['name']}: {call['arguments_chars']} chars so far")
    elif event.type == EphemeralEventType.TOOL_CALL_START:
        tools = event.data["tool_calls"]
        print(f"Calling: {[t['name'] for t in tools]}")
    elif event.type == EphemeralEventType.TOOL_CALL_END:
        print(f"Completed in {event.data.get('duration_ms')}ms")

sub_id = await kit.subscribe_room("room-1", on_tool)
```

### Watching a call being composed

A model calling a tool spends the whole composition of its arguments producing
tokens. For a large argument — a document, an SVG, base64 — that is minutes
during which `TOOL_CALL_START` has not fired yet and nothing else reaches the
bus either. `TOOL_CALL_DELTA` fills that gap:

```json
{
  "tool_calls": [{"id": "tc1", "name": "publish_artifact", "arguments_chars": 3218}],
  "round": 0,
  "channel_id": "ai-agent"
}
```

`arguments_chars` is cumulative per call, so a frame is the round's snapshot and
a dropped one costs nothing. A client that ignores the event loses nothing
either — everything it carries also arrives, complete, in `TOOL_CALL_START`.

**It never carries the argument content.** Only the tool's name and how much has
been composed. Arguments can be megabytes or hold personal data, and
`TOOL_CALL_START` already delivers them in full once the call is complete.

The first fragment of a call publishes immediately — the tool's name is the
signal — and the rest are batched on the same windows as thinking deltas
(`thinking_coalesce_ms`, `thinking_coalesce_chars` on `AIChannel`). Providers
that deliver whole tool calls (Gemini, Ollama) never emit it, and neither does
the non-streaming path.

**An empty `tool_calls` is the terminal frame** — nothing is being composed any
more:

```json
{"tool_calls": [], "round": 0, "channel_id": "ai-agent"}
```

Clear the indicator on it. It always arrives, including on attempts that never
reach `TOOL_CALL_START` at all: cancelled mid-composition, out of tool rounds,
out of time, handed to an external tool provider, failed by the provider,
retried, or replaced by a fallback provider. A retry starts a fresh composition,
so its cumulative character counts do not include the failed attempt. Without
the terminal frame a host would simply be stuck on "composing" instead of stuck
on "working", which is the defect the event exists to remove.

Use these to display "AI is searching..." indicators in a chat UI.

## Custom Ephemeral Events

For application-specific events:

```python
from __future__ import annotations

from roomkit.realtime import EphemeralEvent, EphemeralEventType

event = EphemeralEvent(
    room_id="room-1",
    type=EphemeralEventType.CUSTOM,
    user_id="system",
    data={"action": "file_uploaded", "filename": "report.pdf"},
)
await kit.realtime.publish_to_room("room-1", event)
```

---

## Distributed Deployments: RedisRealtimeBackend

`InMemoryRealtime` only reaches subscribers in the same process. When rooms
are served by multiple workers, use `RedisRealtimeBackend` — ephemeral events
cross process boundaries via Redis pub/sub:

```python
from roomkit import RoomKit
from roomkit.realtime import RedisRealtimeBackend

kit = RoomKit(realtime=RedisRealtimeBackend("redis://localhost:6379"))
```

Requires `pip install roomkit[redis]`. Options:

```python
RedisRealtimeBackend(
    "redis://localhost:6379",
    channel_prefix="roomkit:realtime",  # Redis channel namespace
    max_queue_size=100,                 # per-subscription queue (as InMemoryRealtime)
)
```

Behavior to know:

- **Fire-and-forget** — Redis pub/sub has no replay: events published while a
  worker is disconnected are lost. That matches the ephemeral contract
  (typing and presence are safe to drop); on connection loss the backend
  retries and re-subscribes automatically.
- **Asynchronous local delivery** — subscribers in the publishing process
  receive events through the Redis round-trip, not synchronously during
  `publish()` as with `InMemoryRealtime`.
- **Slow-subscriber isolation** — each subscription keeps its own bounded
  queue and drain task (same mechanism as `InMemoryRealtime`), so one slow
  callback never stalls the shared reader.

See `examples/realtime_redis.py` for a two-terminal cross-process demo, and
[Scaling out](scaling.md) for the full multi-worker picture.

## RealtimeBackend ABC

To plug in another transport (NATS, an SFU data channel, ...), implement the
`RealtimeBackend` ABC — `publish()`, `subscribe()`, `unsubscribe()`, and
optionally `close()`. `EphemeralEvent.to_dict()` and
`EphemeralEvent.from_dict()` handle JSON serialization for transport.

## InMemoryRealtime Details

The default `InMemoryRealtime` is designed for single-process deployments:

| Property | Detail |
|----------|--------|
| **Queue overflow** | LRU-style — oldest events dropped when queue fills (default: 100) |
| **Background tasks** | Each subscription has its own asyncio task |
| **Error resilience** | Callback errors logged, subscription continues |
| **Cleanup** | `close()` cancels all tasks. `RoomKit.close()` calls this |

Adjust the queue size:

```python
from roomkit.realtime import InMemoryRealtime

kit = RoomKit(realtime=InMemoryRealtime(max_queue_size=500))
```

## Combining Ephemeral and Persistent

A typical pattern uses both for responsiveness and durability:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.realtime import EphemeralEvent, EphemeralEventType


async def handle_event(kit: RoomKit, room_id: str, event: EphemeralEvent) -> None:
    if event.type == EphemeralEventType.READ_RECEIPT:
        # Ephemeral: update UI immediately
        await send_to_websocket({"type": "seen", "user": event.user_id})
        # Persistent: also store the read state
        await kit.mark_read(room_id, f"ws-{event.user_id}", event.data["event_id"])
```
