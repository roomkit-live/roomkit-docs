# Delivery Service

`kit.deliver()` sends content to a room's transport channel with awareness of channel state — voice playback, user speech, idle detection. It's the framework-level API for proactive content delivery.

## Quick start

```python
from roomkit import RoomKit, WaitForIdle

kit = RoomKit(delivery_strategy=WaitForIdle(buffer=3.0))

# Deliver content to a room
await kit.deliver("room-id", content="Your payment was confirmed.")
```

## Use cases

- **Delegation results** — workers finish in the background, results delivered to user
- **External events** — webhook arrives, voice agent mentions it
- **Scheduled notifications** — timer fires, agent speaks
- **Cross-room results** — something in room B relevant to room A

## Strategies

Strategies control **when** content is delivered:

```python
from roomkit import Immediate, WaitForIdle, Queued

# Send immediately — may interrupt ongoing voice playback
kit = RoomKit(delivery_strategy=Immediate())

# Wait for AI + user silence, then deliver after buffer
kit = RoomKit(delivery_strategy=WaitForIdle(buffer=3.0))

# Batch multiple deliveries into one message at next idle window
kit = RoomKit(delivery_strategy=Queued(buffer=2.0, separator="\n\n"))
```

| Strategy | When it delivers | Best for |
|----------|-----------------|----------|
| `Immediate()` | Now | Urgent alerts, text channels |
| `WaitForIdle(buffer)` | After AI stops speaking + user stops talking + buffer | Voice conversations |
| `Queued(buffer, separator)` | Batches multiple items, delivers at next idle | High-frequency results |

String shorthand:

```python
await kit.deliver("room", content="hello", strategy="immediate")
await kit.deliver("room", content="hello", strategy="wait_for_idle")
await kit.deliver("room", content="hello", strategy="queued")
```

### WaitForIdle details

`WaitForIdle` is voice-aware:

- **VoiceChannel**: waits for `wait_playback_done()` (TTS finished) + buffer
- **RealtimeVoiceChannel**: waits for `wait_idle()` (provider done + user silent) + buffer
- **Text channels**: delivers immediately (no playback to wait for)

```python
WaitForIdle(
    buffer=3.0,            # seconds to wait after idle detected
    playback_timeout=15.0, # max seconds to wait for playback
)
```

## Channel-aware delivery

`kit.deliver()` auto-detects the best transport channel in the room:

1. **Voice channels** preferred (most latency-sensitive)
2. **RealtimeVoiceChannel** — injects via `inject_text()`
3. **VoiceChannel** — synthetic inbound message → TTS
4. **Other transports** (WebSocket, SMS, etc.) — synthetic inbound message

Override with `channel_id`:

```python
await kit.deliver("room", content="hello", channel_id="voice-main")
```

## Framework default

Set the default strategy on `RoomKit`:

```python
kit = RoomKit(delivery_strategy=WaitForIdle(buffer=3.0))

# All deliver() calls use WaitForIdle unless overridden
await kit.deliver("room", content="result")

# Override per call
await kit.deliver("room", content="urgent!", strategy=Immediate())
```

## Hooks

```python
from roomkit import HookTrigger, HookExecution

@kit.hook(HookTrigger.BEFORE_DELIVER, execution=HookExecution.ASYNC)
async def before_deliver(event, ctx):
    strategy = event.metadata.get("strategy")
    channel = event.metadata.get("channel_id")
    print(f"Delivering via {strategy} to {channel}")

@kit.hook(HookTrigger.AFTER_DELIVER, execution=HookExecution.ASYNC)
async def after_deliver(event, ctx):
    error = event.metadata.get("error")
    if error:
        print(f"Delivery failed: {error}")
    else:
        print("Delivered successfully")
```

| Hook | When | Metadata |
|------|------|----------|
| `BEFORE_DELIVER` | Before strategy executes | `channel_id`, `strategy` |
| `AFTER_DELIVER` | After delivery completes/fails | `channel_id`, `strategy`, `error` |

## Integration with orchestration

The `Supervisor` strategy uses `kit.deliver()` internally:

- **Sync mode** (`async_delivery=False`): results returned inline, no delivery needed
- **Async mode** (`async_delivery=True`): workers run in background, results delivered via `kit.deliver()` when the conversation is idle

```python
from roomkit import RoomKit, Supervisor, WaitForIdle

kit = RoomKit(
    delivery_strategy=WaitForIdle(buffer=3.0),
    orchestration=Supervisor(
        supervisor=coordinator,
        workers=[analyst_1, analyst_2],
        strategy="parallel",
        auto_delegate=True,
        async_delivery=True,
    ),
)
```

See the [Orchestration guide](orchestration.md) for full Supervisor documentation.

## Persistent delivery backends

By default, `kit.deliver()` executes in-process — if the process crashes, pending deliveries are lost. For production deployments, configure a **delivery backend** to decouple enqueue from execution:

```python
from roomkit import RoomKit, InMemoryDeliveryBackend, WaitForIdle

# In-memory backend (single process, no persistence)
kit = RoomKit(
    delivery_strategy=WaitForIdle(buffer=3.0),
    delivery_backend=InMemoryDeliveryBackend(),
)

async with kit:
    await kit.deliver("room", content="Background result ready.")
    # Item is enqueued → worker loop executes delivery asynchronously
```

### How it works

When a `delivery_backend` is configured:

1. `kit.deliver()` serializes the request into a `DeliveryItem` and calls `backend.enqueue()`
2. A background worker loop calls `backend.dequeue()` to claim items
3. The worker deserializes the strategy and executes `strategy.deliver()`
4. On success → `backend.ack()`; on failure → `backend.nack()` (retries or dead-letters)

```
kit.deliver()
  → serialize strategy + content → DeliveryItem
  → backend.enqueue(item)
  → return (non-blocking)

Worker loop (background):
  → backend.dequeue() → claim items
  → BEFORE_DELIVER hook
  → strategy.deliver(ctx)
  → AFTER_DELIVER hook
  → backend.ack() or backend.nack()
```

### Redis backend

For multi-worker deployments, use `RedisDeliveryBackend` with Redis Streams:

```python
from roomkit import RoomKit, WaitForIdle
from roomkit.delivery import RedisDeliveryBackend

kit = RoomKit(
    delivery_strategy=WaitForIdle(buffer=3.0),
    delivery_backend=RedisDeliveryBackend("redis://localhost:6379"),
)
```

Requires `pip install roomkit[redis]`.

Features:

- **Consumer groups** distribute items across workers automatically
- **At-least-once delivery** via Redis Streams PEL (Pending Entries List)
- **Bounded dead-letter stream** for items that exhaust retries
- **Injected client support** for connection pooling

```python
import redis.asyncio as redis

pool = redis.ConnectionPool.from_url("redis://localhost:6379")
client = redis.Redis(connection_pool=pool)

backend = RedisDeliveryBackend(
    client=client,
    stream_prefix="myapp:delivery",
    group_name="myapp-workers",
    max_dead_letter_size=10_000,
)
```

### Available backends

| Backend | Persistence | Multi-worker | Install |
|---------|------------|-------------|---------|
| `InMemoryDeliveryBackend` | No | No | Built-in |
| `RedisDeliveryBackend` | Yes | Yes | `roomkit[redis]` |

### Retry and dead-letter

Failed deliveries are retried up to `max_retries` (default 3). After exhaustion, items move to the dead-letter queue:

```python
# Inspect dead-lettered items
dead = await backend.get_dead_letter_items(limit=50)
for item in dead:
    print(f"{item.id}: {item.error}")

# Check queue depth
depth = await backend.get_queue_depth()
```

### Backward compatibility

If no `delivery_backend` is configured, `kit.deliver()` works exactly as before — in-process with `BEFORE_DELIVER`/`AFTER_DELIVER` hooks.
