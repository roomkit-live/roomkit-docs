# Production Resilience

RoomKit provides built-in patterns for production deployments: circuit breakers, retry policies, rate limiting, delivery tracking, and room lifecycle timers.

## Circuit Breaker

Prevents cascading failures by fast-failing deliveries to an unhealthy
provider. RoomKit keeps one circuit breaker per transport channel, shared by
every room the channel is attached to:

```
CLOSED (normal)
  │ 5 consecutive failed deliveries
  ▼
OPEN (fast-fail)
  │ 60 seconds elapsed
  ▼
HALF_OPEN (testing)
  ├─ the probe delivery succeeds → CLOSED
  └─ it fails → OPEN
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation. A failed delivery, once its binding's retries are spent, increments the counter; a success resets it |
| **OPEN** | Deliveries to the channel are skipped without calling the provider |
| **HALF_OPEN** | One probe delivery goes through. Success closes the breaker, failure reopens it |

A delivery refused for having no recipient does not count as a failure. The
thresholds are fixed; each transition is a framework event:

```python
@kit.on("circuit_breaker_opened")
async def breaker_opened(event):
    logger.warning("Deliveries to %s suspended", event.data["channel_id"])

@kit.on("circuit_breaker_closed")
async def breaker_closed(event):
    logger.info("Deliveries to %s resumed", event.data["channel_id"])
```

## Retry Policy

Exponential backoff with configurable limits:

```python
from __future__ import annotations

from roomkit.models import RetryPolicy

policy = RetryPolicy(
    max_retries=3,
    base_delay_seconds=1.0,
    exponential_base=2.0,
    max_delay_seconds=30.0,
)
```

**Delay formula**: `min(base_delay * exponential_base^attempt, max_delay)`

| Attempt | Delay |
|---------|-------|
| 1 | 1.0s |
| 2 | 2.0s |
| 3 | 4.0s |
| 4 | 8.0s (capped at max_delay) |

Used by `AIChannel` for provider calls (see [AI Steering guide](ai-steering.md) for fallback chains) and the delivery layer for transport providers.

!!! tip
    Only retryable errors trigger retries: a 408, 409, 429 or any 5xx (an
    overload's 529 included), and a transport failure (a connection refused,
    reset or timed out, before the status or while a stream that has emitted
    nothing is read), on every text provider. The status is the one the
    server described, in its status line or in the error event of a stream
    already answered 200, so the same failure reads the same in both forms.
    Any other status fails immediately, and so does a body that is not the
    provider's format (a gateway's HTML page): a `ProviderError`, never an
    empty answer.
    The channel retries a turn's stream; a direct `generate()` call reads
    the same `retryable` flag and retries nothing itself. The provider SDKs
    retry nothing either (`max_retries=0` on the OpenAI, Anthropic, Ollama
    and PolarGrid configs), so one try is one HTTP attempt.

## Connect vs Read Timeout

Every HTTP provider config carries two timeouts, and its adapter hands the
client an `httpx.Timeout` built from both rather than a bare float:

```python
from __future__ import annotations

from roomkit.providers.polargrid import PolarGridConfig

config = PolarGridConfig(
    api_key="pg_...",
    timeout=240.0,  # read/write/pool: sized for the slowest generation
    connect_timeout=5.0,  # TCP connect alone (the default)
)
```

`timeout` is the read budget — how long a slow model may take before the
first byte and between chunks. `connect_timeout` bounds the TCP connect
alone: a host that no longer accepts connections is given up on in seconds,
not after the read budget. Passing one float for both (what a bare
`httpx.AsyncClient(timeout=240.0)` does) never trips on a dead host, because
the kernel exhausts its SYN retries first — about two minutes per attempt,
whatever the value.

The default `connect_timeout` is 5 s, the same as the OpenAI and Anthropic
SDKs' own. It applies to every httpx-backed config in `roomkit.providers`:
the AI providers (OpenAI and its derivatives, Azure, Anthropic, Gemini and
Gemini on Vertex, Ollama, vLLM, PolarGrid), the image providers, and the SMS,
RCS, email, Telegram, Messenger and webhook transports. `AnamConfig.timeout`
has no counterpart because it bounds the SDK's connect call itself, not an
HTTP client.

Outside `roomkit.providers`, the same pair sits on `GrokTTSConfig`,
`OpenAIVisionConfig`, `GeminiVisionConfig`, `GeminiTTSConfig` and
`GeminiSTTConfig`, and is a keyword on `WebSocketAvatarProvider` and
`SSESource`. Two of them read differently:

- `SSESource.timeout` bounds write and pool only. Its read side is left
  unbounded on purpose, so the stream survives idle periods between events;
  `connect_timeout` is what gives up on a dead host.
- The Gemini providers (chat, Vertex, image, TTS, STT, vision) hand the SDK
  their own httpx client (through `HttpOptions.httpx_async_client`) rather
  than a per-request timeout: google-genai flattens a per-request
  `httpx.Timeout` to its largest value, which would hand the read budget
  back to the connect. That client also keeps every SDK call on httpx when
  aiohttp is installed, and carries a request hook: the SDK's classic
  request path (streamed generation, model listing, the Files API) builds
  its request with `timeout=None`, which httpx would read as no timeout at
  all, so the hook puts the client's budget back on a request that names
  none; only the Interactions API path leaves the client default in place.
  `GeminiConfig.timeout` (60 s) is therefore the budget between chunks of a
  streamed answer, not for the whole of it. The STT provider's Files API
  calls set a per-call `timeout`, one value in milliseconds the SDK spreads
  over the connect too; the hook caps that connect at `connect_timeout`, so
  they carry the flat read budget and the client's connect.

!!! tip
    With a `RetryPolicy` of four attempts, a provider whose host is down now
    fails in about 20 s instead of 9 minutes.

## Rate Limiting

A binding's `RateLimit` paces delivery to its channel (a burst waits, nothing
is dropped), and `inbound_rate_limit` drops the inbound messages past its rate
before any processing:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.models import RateLimit

kit = RoomKit(inbound_rate_limit=RateLimit(max_per_second=50.0))

await kit.attach_channel(
    "room-1",
    "sms",
    rate_limit=RateLimit(max_per_second=10.0, max_per_minute=300.0),
)
```

| Parameter | Description |
|-----------|-------------|
| `requests_per_second` | Sustained rate limit |
| `burst_size` | Maximum burst above sustained rate |

The token bucket algorithm allows short bursts up to `burst_size` while enforcing the average rate over time. When the limit is exceeded, inbound messages are rejected before processing.

## Delivery Status Tracking

Providers report what became of an outbound message (sent, delivered, failed) through status webhooks. `kit.process_webhook()` hands each one to `kit.process_delivery_status()`, which runs the `ON_DELIVERY_STATUS` hooks with the room's context. A status is reported, not acted on: RoomKit does not wait for it, and a `failed` status does not resend the message.

!!! note "Delivery guarantees per binding are not available yet"
    A binding cannot require a delivery confirmation or an acknowledgement
    from the recipient; that is a planned feature. Retries today cover the
    send itself (a binding's `retry_policy`, with the backoff above). React
    to a failed status in the hook below.

### Delivery Status Hook

```python
@kit.hook(HookTrigger.ON_DELIVERY_STATUS)
async def on_delivery(event, ctx):
    # event is a DeliveryStatus: provider, message_id, status, recipient, timestamp
    if event.status == "failed":
        logger.error("Delivery to %s failed: %s", event.recipient, event.error_message)
    elif event.status == "delivered":
        logger.info("Delivered to %s on %s", event.recipient, event.channel_id)
```

## Room Lifecycle Timers

Drive room state automatically based on inactivity — pause after a short idle period, close after a longer one:

```python
from __future__ import annotations

from roomkit import RoomKit, RoomTimers

kit = RoomKit()

room = await kit.create_room(
    room_id="support-123",
    timers=RoomTimers(
        inactive_after_seconds=300,    # Pause after 5 min without activity
        closed_after_seconds=3600,     # Close after 1 hour paused
    ),
)
```

| Timer | Effect |
|-------|--------|
| `inactive_after_seconds` | Pauses an ACTIVE room after N seconds without activity (from `last_activity_at`, or `created_at` before any) |
| `closed_after_seconds` | Closes a PAUSED room N seconds after its `paused_at` |

Every operation that reads the room's status under its lock applies the transitions due at its instant first, and a delivered message resumes a paused room. A room nobody writes to transitions when `kit.check_room_timers(room_id)` or `kit.check_all_timers()` runs — RoomKit has no internal scheduler, so run the sweep from a background task. Each transition takes the instant it fell due, emits `room_paused` / `room_closed` (`reason = timer`), and fires the `ON_ROOM_PAUSED` / `ON_ROOM_CLOSED` lifecycle hooks once the room lock is released.

See the [Room Lifecycle & Timers guide](room-lifecycle.md) for statuses, the activity-tracking model, driving the sweep, and resuming paused rooms.

## Complete Production Setup

Combining all patterns for a resilient deployment:

```python
from __future__ import annotations

from roomkit import RoomKit, RoomTimers
from roomkit.channels import AIChannel, SMSChannel
from roomkit.models import RateLimit, RetryPolicy
from roomkit.providers.anthropic import AnthropicAIProvider
from roomkit.providers.openai import OpenAIAIProvider
from roomkit.store import PostgresStore
from roomkit.telemetry import OpenTelemetryProvider

# Production store + telemetry
store = PostgresStore("postgresql://user:pass@db/roomkit")
telemetry = OpenTelemetryProvider(service_name="roomkit-prod")

kit = RoomKit(
    store=store,
    inbound_rate_limit=RateLimit(max_per_second=50.0),
    telemetry=telemetry,
)

# AI with fallback and retries
primary = AnthropicAIProvider(model="claude-opus-5", api_key="...")
fallback = OpenAIAIProvider(model="gpt-4o", api_key="...")

ai = AIChannel(
    "ai-assistant",
    provider=primary,
    fallback_provider=fallback,
    retry_policy=RetryPolicy(max_retries=3, base_delay_seconds=1.0),
)

sms = SMSChannel("sms-main", provider=twilio_provider)

kit.register_channel(ai)
kit.register_channel(sms)

# Room with lifecycle timers
room = await kit.create_room(
    room_id="support-123",
    timers=RoomTimers(inactive_after_seconds=300, closed_after_seconds=3600),
)

# SMS delivery paced by the binding
await kit.attach_channel(
    "support-123",
    "sms-main",
    rate_limit=RateLimit(max_per_second=10.0),
)
```

## Health Monitoring

Use telemetry spans and metrics to monitor resilience patterns:

```python
from __future__ import annotations

from roomkit import HookExecution, HookTrigger

@kit.hook(HookTrigger.ON_ERROR, execution=HookExecution.ASYNC)
async def on_error(event, ctx):
    logger.error(f"Error in {event.channel_id}: {event.error}")
    # Emit metric to monitoring system
    metrics.increment("roomkit.errors", tags={"channel": event.channel_id})

@kit.hook(HookTrigger.ON_DELIVERY_STATUS, execution=HookExecution.ASYNC)
async def track_delivery(event, ctx):
    metrics.increment(
        "roomkit.delivery",
        tags={"status": event.status, "channel": event.channel_id},
    )
```

The `OpenTelemetryProvider` automatically instruments all framework operations with spans — monitor via Jaeger, DataDog, or any OTEL-compatible backend. See the [Telemetry guide](telemetry.md) for setup details.
