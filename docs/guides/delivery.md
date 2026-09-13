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

## Observable outcomes

Every call returns a `DeliveryOutcome`:

| Status | Meaning |
|--------|---------|
| `queued` | The delivery backend accepted the item; no transmission is established yet. |
| `sent` | The text event was published, or the realtime provider accepted the injection. |
| `blocked` | A hook, room state or permission refused the request. |
| `unavailable` | A channel, addressed agent or voice session could not be reached. |
| `failed` | Enqueue, strategy, provider or delivery execution failed. |
| `unknown` | A custom strategy executed without returning an outcome. |

Read `reason` for a machine-readable explanation, `error` for structured failure
information and `unavailable_targets` for unresolved addresses. `event_id` can be
present on an unavailable/failed result: publication may succeed while agent
solicitation or delivery fails. A failure after publication is not retried by
the worker as a new text event.

`sent` does not establish that the agent completed its turn. Direct `Immediate`
and `WaitForIdle` calls wait for the existing text delivery handle when doing so
cannot deadlock the room. `Queued` returns at publication, so its drain can accept
requests from tools in an ongoing turn. Its `inbound.delivery` handle can be
awaited separately. `turn_complete` reports known text completion, including
failed completion; it is false for a replay or realtime injection.

```python
result = await kit.deliver(
    "room", "Background result ready", addressed_to=["agent-a"],
    idempotency_key="external-event:42",
)
if result.inbound is not None and result.inbound.delivery is not None:
    turn = await result.inbound.delivery.wait()
    # Inspect turn.error, turn.response_events and turn.response_metadata.
```

The live `inbound` object is excluded from JSON serialization. Persisted queue
items and hook metadata contain the serializable outcome snapshot. A queued
call's `delivery_item_id` links it to the later `AFTER_DELIVER` observation;
there is no promise that the initial queued result updates itself.

## Addressing an external event

```python
result = await kit.deliver(
    "conversation", "The background task finished.",
    channel_id="websocket",       # Transport/source attribution
    addressed_to=["agent-a"],     # Intelligence channel asked to act
    idempotency_key="task-result:42",
    metadata={"event_type": "task.finished", "task_id": "42"},
)
```

`addressed_to=None` retains the room's routing policy. `[]` stores and broadcasts
the event without soliciting an intelligence channel. Unknown or unreadable
addresses are reported as unavailable; another agent is never selected in their
place. Valid addresses in a mixed list can still act. The stored event retains
the whole address, and addressing does not change visibility, permissions or
muting. A configured supervisor retains the RFC's supervisory exception.

Naming an intelligence channel in `channel_id` selects its room's transport for
compatibility. Use `addressed_to` to choose which intelligence channel acts.

### Text idempotency and retries

A key identifies one publication within a room, even if a later call supplies
a different body or address. Concurrent calls with that key return the original
`event_id`; replays set `duplicate=True` and do not run the agent again. The
existing locked inbound pipeline owns this decision. Different keys, rooms, or
calls without a key remain distinct. Empty keys are refused.

Keys last as long as the configured `ConversationStore` retains them. The memory
store loses them on restart; SQLite and Postgres preserve committed keys across
restart until the corresponding event/key is removed. Distributed deployments
need a shared store and an appropriate shared lock manager. Custom stores must
implement the same idempotency contract; an older store that cannot resolve a
seen key may return a duplicate refusal without an event id.

A stored event does not prove its turn completed. If a process crashes after
commit, replaying the key avoids a second trigger but does not resume the lost
turn. Durable delivery-lane outboxes and crash recovery are separate capabilities.

### Selecting a realtime session

```python
result = await kit.deliver(
    "call-room", "Your result is ready.", channel_id="voice-main",
    session_id=session.id, strategy=WaitForIdle(buffer=1.0),
)
```

`session_id` requires an explicit realtime `channel_id` and cannot be combined
with `addressed_to`. Without a session id, an explicitly selected realtime
channel needs exactly one active session; otherwise the result is unavailable.
Idle strategies pin that session before waiting and verify it again at injection.
An ended/replaced session never selects a replacement. For deferred delivery,
provide the original session id: resolution otherwise happens when the worker
executes, not when the producer enqueues.

Unaddressed room-only calls retain realtime fanout. Supplying a non-null
`addressed_to` uses the text pipeline even on a realtime transport; it never
turns that intelligence address into direct session injection. Transport delivery
and visibility still follow the room's ordinary broadcast rules.

Realtime injection has **no idempotency guarantee**. Supplying a key does not
suppress repeated speech; the outcome states `voice_not_deduplicated` and lists
accepted `session_ids`. A provider can accept text and then fail before the
caller receives confirmation. Retries, queue redelivery and restarts can repeat
injection; no exactly-once agent execution or turn completion is promised.
Muted/read-only bindings receive silent context injection; unreadable bindings
are refused.

### Queued grouping

Reuse a `Queued` instance to batch compatible requests. Rooms, transports,
intelligence addresses, session targets and metadata must match. Requests with
idempotency keys publish individually, retaining each key and outcome attribution.
A request arriving during transmission is drained too. Cancelling the drain
wakes waiting callers; it does not roll back committed events.

## Hooks

```python
from roomkit import HookTrigger, HookExecution, HookResult

@kit.hook(HookTrigger.BEFORE_DELIVER)
async def before_deliver(event, ctx):
    # This synchronous gate may return HookResult.block("policy") or modify text.
    return HookResult.allow()

@kit.hook(HookTrigger.AFTER_DELIVER, execution=HookExecution.ASYNC)
async def after_deliver(event, ctx):
    outcome = event.metadata["delivery_outcome"]
    logger.info("Delivery %s: %s", outcome["status"], outcome["reason"])
```

`BEFORE_DELIVER` runs synchronously; an explicit refusal stops delivery, while
hook infrastructure failures follow the RFC's fail-open rule. `AFTER_DELIVER`
observes refusals as well as successful and failed attempts, with effective
rewritten content. Hook observation is best-effort. Neither hook changes the
inbound pipeline ordering or its own broadcast hooks.

| Hook | When | Payload |
|------|------|---------|
| `BEFORE_DELIVER` | Before strategy executes | Address and key on the event; `channel_id`, `strategy`, `delivery_item_id`, `session_id` in metadata |
| `AFTER_DELIVER` | After an execution attempt | Same fields, plus `delivery_outcome` and `error` (null on success/replay) |

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
4. Consume sent/refused items with `ack()`; retry transient unavailable/failed outcomes with `nack()`; dead-letter non-retryable failures.

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

Retryable failures are attempted up to `max_retries` (default 3). Exhausted or non-retryable failures move to the dead-letter queue with `item.outcome`, address, key and session target preserved. Explicit hook refusals are consumed without retry and retain a blocked outcome:

```python
# Inspect dead-lettered items
dead = await backend.get_dead_letter_items(limit=50)
for item in dead:
    print(f"{item.id}: {item.error}")

# Check queue depth
depth = await backend.get_queue_depth()
```

### Backward compatibility

Calls without the new options remain valid. Without a backend, execution stays in-process; callers that ignore the returned outcome can continue doing so. Custom strategies returning `None` execute but report `unknown`.

Run `uv run python examples/external_event_delivery.py` for a network-free example with two agents and a replayed external event.
