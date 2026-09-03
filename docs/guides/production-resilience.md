# Production Resilience

RoomKit provides built-in patterns for production deployments: circuit breakers, retry policies, rate limiting, delivery tracking, and room lifecycle timers.

## Circuit Breaker

Prevents cascading failures by fast-failing requests to unhealthy providers:

```python
from __future__ import annotations

from roomkit.models.delivery import CircuitBreaker

breaker = CircuitBreaker(
    failure_threshold=5,      # Open after 5 consecutive failures
    recovery_timeout=30.0,    # Try again after 30 seconds
    half_open_max=2,          # Allow 2 test requests in half-open state
)
```

### States

```
CLOSED (normal)
  │ failure_threshold consecutive failures
  ▼
OPEN (fast-fail)
  │ recovery_timeout elapsed
  ▼
HALF_OPEN (testing)
  ├─ half_open_max successes → CLOSED
  └─ any failure → OPEN
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation. Failures increment the counter |
| **OPEN** | All requests fail immediately without calling the provider |
| **HALF_OPEN** | Limited test requests allowed. Success resets to CLOSED, failure reopens |

| Parameter | Default | Description |
|-----------|---------|-------------|
| `failure_threshold` | `5` | Consecutive failures before opening |
| `recovery_timeout` | `30.0` | Seconds before trying again |
| `half_open_max` | `2` | Test requests allowed in half-open |

## Retry Policy

Exponential backoff with configurable limits:

```python
from __future__ import annotations

from roomkit.models.channel import RetryPolicy

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
    Only retryable errors (5xx, timeouts) trigger retries. Non-retryable errors (4xx) fail immediately.

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
  aiohttp is installed, and puts the budget back on the streamed request
  the SDK builds with `timeout=None` (the chat providers' only call), which
  httpx would otherwise read as no timeout at all. `GeminiConfig.timeout`
  (60 s) is therefore the budget between chunks of a streamed answer, not
  for the whole of it. The STT provider's Files API calls go through the
  SDK's classic path, which takes one value in milliseconds, so they carry
  the flat `timeout` and no connect split.

!!! tip
    With a `RetryPolicy` of four attempts, a provider whose host is down now
    fails in about 20 s instead of 9 minutes.

## Rate Limiting

Token bucket rate limiting for inbound messages:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.channels import SMSChannel
from roomkit.core.rate_limit import RateLimit, TokenBucketRateLimiter

limiter = TokenBucketRateLimiter()
kit = RoomKit(rate_limiter=limiter)

sms = SMSChannel(
    "sms",
    provider=twilio_provider,
    rate_limit=RateLimit(
        requests_per_second=10,
        burst_size=20,
    ),
)
```

| Parameter | Description |
|-----------|-------------|
| `requests_per_second` | Sustained rate limit |
| `burst_size` | Maximum burst above sustained rate |

The token bucket algorithm allows short bursts up to `burst_size` while enforcing the average rate over time. When the limit is exceeded, inbound messages are rejected before processing.

## Delivery Status Tracking

Track message delivery through three modes:

```python
from __future__ import annotations

from roomkit import HookTrigger, RoomKit
from roomkit.models.delivery import DeliveryMode

kit = RoomKit()

# Configure delivery mode per channel binding
await kit.attach_channel("room-1", "sms-main", metadata={
    "delivery_mode": DeliveryMode.REQUIRE_DELIVERY,
})
```

| Mode | Behavior |
|------|----------|
| `FIRE_AND_FORGET` | Send and don't wait for confirmation |
| `REQUIRE_DELIVERY` | Track delivery status, retry on failure |
| `REQUIRE_ACK` | Require acknowledgement from recipient |

### Delivery Status Hook

```python
@kit.hook(HookTrigger.ON_DELIVERY_STATUS)
async def on_delivery(event, ctx):
    # event contains DeliveryStatus with provider_id, status, timestamp
    if event.status == "failed":
        logger.error(f"Delivery failed to {event.channel_id}: {event.error}")
    elif event.status == "delivered":
        logger.info(f"Delivered to {event.channel_id}")
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
        inactive_after_seconds=300,    # Pause after 5 min idle
        closed_after_seconds=3600,     # Close after 1 hour idle
    ),
)
```

| Timer | Effect |
|-------|--------|
| `inactive_after_seconds` | Pauses the room after N seconds without activity |
| `closed_after_seconds` | Closes the room after N seconds without activity (takes priority over pause) |

Transitions are evaluated on demand via `kit.check_room_timers(room_id)` or `kit.check_all_timers()` — RoomKit has no internal scheduler, so run the sweep from a background task. Each transition fires the `ON_ROOM_PAUSED` / `ON_ROOM_CLOSED` lifecycle hooks.

See the [Room Lifecycle & Timers guide](room-lifecycle.md) for statuses, the activity-tracking model, driving the sweep, and resuming paused rooms.

## Complete Production Setup

Combining all patterns for a resilient deployment:

```python
from __future__ import annotations

from roomkit import RoomKit, RoomTimers
from roomkit.channels import AIChannel, SMSChannel
from roomkit.core.rate_limit import RateLimit, TokenBucketRateLimiter
from roomkit.models.channel import RetryPolicy
from roomkit.providers.ai.anthropic import AnthropicAIProvider
from roomkit.providers.ai.openai import OpenAIAIProvider
from roomkit.store.postgres import PostgresStore
from roomkit.telemetry import OpenTelemetryProvider

# Production store + telemetry
store = PostgresStore("postgresql://user:pass@db/roomkit")
telemetry = OpenTelemetryProvider(service_name="roomkit-prod")

kit = RoomKit(
    store=store,
    rate_limiter=TokenBucketRateLimiter(),
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

# SMS with rate limiting
sms = SMSChannel(
    "sms-main",
    provider=twilio_provider,
    rate_limit=RateLimit(requests_per_second=10, burst_size=20),
)

kit.register_channel(ai)
kit.register_channel(sms)

# Room with lifecycle timers
room = await kit.create_room(
    room_id="support-123",
    timers=RoomTimers(inactive_after_seconds=300, closed_after_seconds=3600),
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
