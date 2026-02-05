# Realtime Events

RoomKit's realtime system handles **ephemeral events** - transient signals that don't require persistence like typing indicators, presence updates, and read receipts.

## Overview

Ephemeral events are published to rooms and delivered to subscribers in real-time. Unlike `RoomEvent`s, they are not stored in the conversation history.

```python
from roomkit import RoomKit, EphemeralEvent, EphemeralEventType

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
from roomkit import EphemeralEvent, EphemeralEventType

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
from roomkit import RoomKit, EphemeralEvent, EphemeralEventType

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

## Custom Backends

By default, RoomKit uses `InMemoryRealtime` which works for single-process deployments. For distributed systems, implement a custom `RealtimeBackend`:

```python
from roomkit import RealtimeBackend, EphemeralEvent, EphemeralCallback
import redis.asyncio as redis
import json

class RedisRealtime(RealtimeBackend):
    def __init__(self, url: str = "redis://localhost:6379"):
        self._redis = redis.from_url(url)
        self._pubsub = self._redis.pubsub()
        self._subscriptions: dict[str, tuple[str, EphemeralCallback]] = {}

    async def publish(self, channel: str, event: EphemeralEvent) -> None:
        await self._redis.publish(channel, json.dumps({
            "id": event.id,
            "room_id": event.room_id,
            "type": event.type.value,
            "user_id": event.user_id,
            "channel_id": event.channel_id,
            "data": event.data,
            "timestamp": event.timestamp.isoformat(),
        }))

    async def subscribe(self, channel: str, callback: EphemeralCallback) -> str:
        from uuid import uuid4
        sub_id = uuid4().hex
        await self._pubsub.subscribe(channel)
        self._subscriptions[sub_id] = (channel, callback)
        # Start listener task...
        return sub_id

    async def unsubscribe(self, subscription_id: str) -> bool:
        if subscription_id not in self._subscriptions:
            return False
        channel, _ = self._subscriptions.pop(subscription_id)
        await self._pubsub.unsubscribe(channel)
        return True

    async def close(self) -> None:
        await self._pubsub.close()
        await self._redis.close()

# Use custom backend
kit = RoomKit(realtime=RedisRealtime("redis://localhost:6379"))
```

---

## API Reference

::: roomkit.EphemeralEventType

::: roomkit.EphemeralEvent

::: roomkit.EphemeralCallback

::: roomkit.RealtimeBackend

::: roomkit.InMemoryRealtime
