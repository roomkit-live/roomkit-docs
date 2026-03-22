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
