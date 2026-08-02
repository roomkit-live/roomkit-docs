# Realtime Events

RoomKit's realtime system handles **ephemeral events** - transient signals that don't require persistence like typing indicators, presence updates, and read receipts.

## Overview

Ephemeral events are published to rooms and delivered to subscribers in real-time. Unlike `RoomEvent`s, they are not stored in the conversation history.

```python
from roomkit import RoomKit
from roomkit.realtime.base import EphemeralEvent, EphemeralEventType

kit = RoomKit()

# Subscribe to ephemeral events
async def handle_ephemeral(event: EphemeralEvent):
    if event.type == EphemeralEventType.TYPING_START:
        print(f"{event.user_id} is typing...")

sub_id = await kit.subscribe_room("room-123", handle_ephemeral)

# Publish typing indicator
await kit.publish_typing("room-123", "user-456")
```

## Typing Indicators

Notify participants when someone is typing.

```python
# Start typing
await kit.publish_typing(room_id="room-123", user_id="user-456", is_typing=True)

# Stop typing (explicit)
await kit.publish_typing(room_id="room-123", user_id="user-456", is_typing=False)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `room_id` | `str` | required | The room to publish the typing event in |
| `user_id` | `str` | required | The user who is typing |
| `is_typing` | `bool` | `True` | `True` for `TYPING_START`, `False` for `TYPING_STOP` |

## Presence Updates

Track user availability in a room.

```python
# User comes online
await kit.publish_presence(room_id="room-123", user_id="user-456", status="online")

# User goes away
await kit.publish_presence(room_id="room-123", user_id="user-456", status="away")

# User goes offline
await kit.publish_presence(room_id="room-123", user_id="user-456", status="offline")
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `room_id` | `str` | The room to publish the presence event in |
| `user_id` | `str` | The user whose presence changed |
| `status` | `str` | One of: `"online"`, `"away"`, `"offline"` |

## Read Receipts

Indicate that a user has read messages up to a specific event.

```python
await kit.publish_read_receipt(
    room_id="room-123",
    user_id="user-456",
    event_id="evt-789"  # The last event the user has read
)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `room_id` | `str` | The room containing the event |
| `user_id` | `str` | The user who read the message |
| `event_id` | `str` | ID of the last read event |

## Subscribing to Events

Subscribe to receive ephemeral events for a room.

```python
from roomkit.realtime.base import EphemeralEvent, EphemeralEventType

async def handle_ephemeral(event: EphemeralEvent):
    match event.type:
        case EphemeralEventType.TYPING_START:
            print(f"{event.user_id} started typing")
        case EphemeralEventType.TYPING_STOP:
            print(f"{event.user_id} stopped typing")
        case EphemeralEventType.PRESENCE_ONLINE:
            print(f"{event.user_id} came online")
        case EphemeralEventType.PRESENCE_AWAY:
            print(f"{event.user_id} went away")
        case EphemeralEventType.PRESENCE_OFFLINE:
            print(f"{event.user_id} went offline")
        case EphemeralEventType.READ_RECEIPT:
            print(f"{event.user_id} read up to {event.data.get('event_id')}")

# Subscribe - returns a subscription ID
subscription_id = await kit.subscribe_room("room-123", handle_ephemeral)

# Later: unsubscribe when done
await kit.unsubscribe_room(subscription_id)
```

## WebSocket Integration Example

Complete example forwarding ephemeral events to WebSocket clients:

```python
from fastapi import FastAPI, WebSocket
from roomkit import RoomKit
from roomkit.realtime.base import EphemeralEvent, EphemeralEventType

app = FastAPI()
kit = RoomKit()

@app.websocket("/ws/{room_id}/{user_id}")
async def websocket_endpoint(ws: WebSocket, room_id: str, user_id: str):
    await ws.accept()

    # Forward ephemeral events to the WebSocket client
    async def forward_ephemeral(event: EphemeralEvent):
        await ws.send_json({
            "type": "ephemeral",
            "event_type": event.type.value,
            "user_id": event.user_id,
            "room_id": event.room_id,
            "timestamp": event.timestamp.isoformat(),
            "data": event.data,
        })

    # Subscribe to room's ephemeral events
    sub_id = await kit.subscribe_room(room_id, forward_ephemeral)

    try:
        while True:
            data = await ws.receive_json()

            if data.get("action") == "typing":
                await kit.publish_typing(
                    room_id,
                    user_id=user_id,
                    is_typing=data.get("is_typing", True),
                )
            elif data.get("action") == "presence":
                await kit.publish_presence(
                    room_id,
                    user_id=user_id,
                    status=data.get("status", "online"),
                )
            elif data.get("action") == "read":
                await kit.publish_read_receipt(
                    room_id,
                    user_id=user_id,
                    event_id=data.get("event_id"),
                )
    finally:
        # Clean up subscription on disconnect
        await kit.unsubscribe_room(sub_id)
```

## Distributed Backends

By default, RoomKit uses `InMemoryRealtime` which works for single-process
deployments. For distributed systems, `RedisRealtimeBackend` ships with the
library (requires `pip install roomkit[redis]`):

```python
from roomkit import RoomKit
from roomkit.realtime import RedisRealtimeBackend

kit = RoomKit(realtime=RedisRealtimeBackend("redis://localhost:6379"))
```

See the [Realtime Features guide](../guides/realtime-features.md#distributed-deployments-redisrealtimebackend)
for semantics (fire-and-forget, slow-subscriber isolation). For another
transport (NATS, ...), implement the `RealtimeBackend` ABC below.

---

## API Reference

::: roomkit.realtime.base.EphemeralEventType

::: roomkit.realtime.base.EphemeralEvent

::: roomkit.realtime.base.EphemeralCallback

::: roomkit.realtime.base.RealtimeBackend

::: roomkit.realtime.memory.InMemoryRealtime

::: roomkit.realtime.redis.RedisRealtimeBackend
