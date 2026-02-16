# Telemetry

RoomKit includes a provider-agnostic telemetry system for tracing spans and recording metrics across voice, AI, and framework operations. The system follows RoomKit's ABC pattern -- plug in OpenTelemetry, Langfuse, Logfire, or any custom backend.

## Quick Start

```python
from roomkit import RoomKit
from roomkit.telemetry import ConsoleTelemetryProvider

kit = RoomKit(telemetry=ConsoleTelemetryProvider())
```

This logs span summaries and metrics to Python's `logging` module. For production, use the OpenTelemetry provider or implement your own.

## Architecture

The telemetry system has four layers:

1. **TelemetryProvider ABC** -- defines span and metric operations
2. **Built-in providers** -- Noop (default), Console, Mock, OpenTelemetry
3. **Instrumentation points** -- automatic spans in STT, TTS, LLM, hooks, pipeline, delivery, store, and voice backends
4. **Wiring** -- RoomKit propagates the provider to all channels, the hook engine, the event router, the store, and AI/STT/TTS providers

### Span Hierarchy

Spans form parent-child trees using a context variable (`contextvars.ContextVar`) for automatic propagation. A typical voice conversation shows as a single trace in Jaeger:

```
VOICE_SESSION [session_id=abc123, room_id=room-1, backend=FastRTCBackend]
  └─ STT_TRANSCRIBE [session_id=abc123, provider=deepgram]
  └─ INBOUND_PIPELINE [room_id=room-1, session_id=abc123]
       └─ HOOK_SYNC (before_broadcast) [room_id=room-1]
       └─ BROADCAST [room_id=room-1, session_id=abc123]
            └─ LLM_GENERATE [room_id=room-1]
                 └─ LLM_TOOL_CALL [tool=get_weather]
            └─ DELIVERY [delivery.channel_type=sms]
            └─ HOOK_ASYNC (after_broadcast) [room_id=room-1]
  └─ TTS_SYNTHESIZE [session_id=abc123, provider=elevenlabs]
  └─ STT_TRANSCRIBE (next utterance)
  └─ INBOUND_PIPELINE (next turn) ...
```

Non-voice messages (e.g. SMS inbound) produce a flat tree:

```
INBOUND_PIPELINE [room_id=room-1, channel_id=sms-main]
  └─ HOOK_SYNC (before_broadcast) [room_id=room-1]
  └─ BROADCAST [room_id=room-1]
       └─ LLM_GENERATE
       └─ DELIVERY
       └─ HOOK_ASYNC (after_broadcast)
```

For realtime voice (Gemini Live, OpenAI Realtime):

```
REALTIME_SESSION [provider=GeminiLiveProvider]
  └─ REALTIME_TURN
       └─ REALTIME_TOOL_CALL [tool=get_weather]
  └─ REALTIME_TURN
```

For store operations:

```
STORE_QUERY [store.operation=append_event, store.table=events]
```

#### Session Correlation

Every span in a voice conversation carries `session_id` and/or `room_id`. In Jaeger or any OTel-compatible backend, you can filter by `session_id=abc123` to see the complete conversation tree.

- **Voice-originated messages**: `_route_text()` sets the voice session span as the parent context before calling `process_inbound()`, so INBOUND_PIPELINE becomes a child of VOICE_SESSION.
- **STT/TTS spans**: Use direct lookup from `_voice_session_spans` to parent under VOICE_SESSION (more reliable than context vars since these run in callbacks).
- **room_id backfill**: INBOUND_PIPELINE is created before room routing completes. The `room_id` is set as an attribute once routing determines the target room.

## Built-in Providers

### NoopTelemetryProvider (Default)

Zero overhead. All methods are no-ops. Used when no telemetry is configured.

```python
kit = RoomKit()  # Uses NoopTelemetryProvider by default
```

### ConsoleTelemetryProvider

Logs span summaries to `logging.getLogger("roomkit.telemetry")`. Useful for development and debugging.

```python
from roomkit.telemetry import ConsoleTelemetryProvider

kit = RoomKit(telemetry=ConsoleTelemetryProvider())
```

Output:

```
INFO  SPAN START stt.transcribe — stt.batch [room=room-1, session=abc123]
INFO  SPAN END   stt.transcribe — stt.batch (234.5ms) {stt.text_length: 42}
INFO  METRIC     roomkit.stt.ttfb_ms = 89.20 ms
```

### MockTelemetryProvider

Records all spans and metrics in lists. Use in tests to assert telemetry behavior.

```python
from roomkit.telemetry import MockTelemetryProvider, SpanKind

mock = MockTelemetryProvider()
kit = RoomKit(telemetry=mock)

# ... run your test ...

stt_spans = mock.get_spans(SpanKind.STT_TRANSCRIBE)
assert len(stt_spans) == 1
assert stt_spans[0].attributes["stt.text_length"] == 42
```

### OpenTelemetryProvider

Bridges to the OpenTelemetry SDK. Requires optional dependencies:

```bash
pip install 'roomkit[opentelemetry]'
```

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))

from roomkit import RoomKit
from roomkit.telemetry.opentelemetry import OpenTelemetryProvider

kit = RoomKit(telemetry=OpenTelemetryProvider(tracer_provider=provider))
```

The provider also supports `meter_provider` for OTel metrics.

## SpanKind Reference

| SpanKind | Where | Description |
|---|---|---|
| `STT_TRANSCRIBE` | `_voice_stt.py` | Batch STT transcription |
| `TTS_SYNTHESIZE` | `_voice_tts.py` | TTS synthesis + delivery |
| `LLM_GENERATE` | `ai.py` | LLM generation including tool loops |
| `LLM_TOOL_CALL` | `ai.py` | Individual tool execution |
| `HOOK_SYNC` | `hooks.py` | Synchronous hook execution |
| `HOOK_ASYNC` | `hooks.py` | Async hook execution |
| `INBOUND_PIPELINE` | `_inbound.py` | Full inbound processing |
| `BROADCAST` | `event_router.py` | Event broadcast to all channels |
| `DELIVERY` | `transport.py` | Transport channel delivery |
| `VOICE_SESSION` | `voice.py` | Voice session lifecycle (start → end) |
| `STORE_QUERY` | `postgres.py` | Database operation (create, get, append) |
| `BACKEND_CONNECT` | `fastrtc.py`, `sip.py`, `rtp.py` | Voice backend connection |
| `REALTIME_SESSION` | `realtime_voice.py` | Realtime voice session lifetime |
| `REALTIME_TURN` | `realtime_voice.py` | Single AI response turn |
| `REALTIME_TOOL_CALL` | `realtime_voice.py` | Realtime tool execution |
| `PIPELINE_INBOUND` | `engine.py` | Audio pipeline metrics |

## Well-Known Attributes

The `Attr` class provides constants to prevent typos:

```python
from roomkit.telemetry import Attr

# Common
Attr.PROVIDER          # "provider"
Attr.MODEL             # "model"
Attr.TTFB_MS           # "ttfb_ms"
Attr.DURATION_MS       # "duration_ms"

# STT
Attr.STT_TEXT_LENGTH   # "stt.text_length"
Attr.STT_CONFIDENCE    # "stt.confidence"

# TTS
Attr.TTS_CHAR_COUNT    # "tts.char_count"
Attr.TTS_VOICE         # "tts.voice"

# LLM
Attr.LLM_INPUT_TOKENS  # "llm.input_tokens"
Attr.LLM_OUTPUT_TOKENS # "llm.output_tokens"

# Realtime
Attr.REALTIME_PROVIDER  # "realtime.provider"
Attr.REALTIME_TOOL_NAME # "realtime.tool_name"

# Delivery
Attr.DELIVERY_CHANNEL_TYPE  # "delivery.channel_type"
Attr.DELIVERY_RECIPIENT     # "delivery.recipient"
Attr.DELIVERY_SUCCESS       # "delivery.success"
Attr.DELIVERY_ERROR         # "delivery.error"
Attr.DELIVERY_MESSAGE_ID    # "delivery.message_id"

# Store
Attr.STORE_OPERATION  # "store.operation"
Attr.STORE_TABLE      # "store.table"

# Backend
Attr.BACKEND_TYPE  # "backend.type"
```

## TelemetryConfig

For advanced configuration, use `TelemetryConfig`:

```python
from roomkit.telemetry import TelemetryConfig, ConsoleTelemetryProvider

config = TelemetryConfig(
    provider=ConsoleTelemetryProvider(),
    sample_rate=0.1,          # Sample 10% of spans
    enabled_spans=None,       # All spans (or pass a set of SpanKind)
    metadata={"env": "prod"}, # Extra metadata added to every span
)

kit = RoomKit(telemetry=config)
```

### Searchable Tags (Jaeger / OTel)

The `metadata` dict adds custom attributes to **every** span, making them searchable as tags in Jaeger. Use this for environment, app, or deployment identifiers:

```python
config = TelemetryConfig(
    provider=OpenTelemetryProvider(tracer_provider=provider),
    metadata={
        "env": "production",
        "app": "my-voice-agent",
        "version": "1.2.3",
    },
)

kit = RoomKit(telemetry=config)
```

In Jaeger, search for `roomkit.env=production` or `roomkit.app=my-voice-agent` to find all spans from that deployment. Session and room identifiers (`roomkit.session_id`, `roomkit.room_id`) are automatically set on spans where available.

You can also pass `metadata` directly to the `OpenTelemetryProvider`:

```python
otel = OpenTelemetryProvider(
    tracer_provider=provider,
    metadata={"env": "staging"},
)
```

## Noise Reduction

High-frequency hooks like audio level monitoring fire ~10 times per second, creating a flood of spans in Jaeger and console output. RoomKit suppresses telemetry spans for these hooks by default.

### Default Suppressed Triggers

The following hook triggers are suppressed by default:

- `on_input_audio_level` -- Microphone input level (~10/sec)
- `on_output_audio_level` -- Speaker output level (~10/sec)
- `on_vad_audio_level` -- VAD energy level (~10/sec)

The hooks still fire normally -- only the telemetry spans are suppressed.

### Custom Suppression

Override the default set via `TelemetryConfig.suppressed_hook_triggers`:

```python
from roomkit.telemetry import TelemetryConfig, ConsoleTelemetryProvider

# Suppress additional hooks
config = TelemetryConfig(
    provider=ConsoleTelemetryProvider(),
    suppressed_hook_triggers={
        "on_input_audio_level",
        "on_output_audio_level",
        "on_vad_audio_level",
        "on_speech_start",  # Also suppress speech start spans
    },
)

kit = RoomKit(telemetry=config)
```

To enable spans for all hooks (no suppression):

```python
config = TelemetryConfig(
    provider=ConsoleTelemetryProvider(),
    suppressed_hook_triggers=set(),  # Empty set = no suppression
)
```

## Custom Provider

Implement `TelemetryProvider` to integrate with any backend:

```python
from roomkit.telemetry.base import SpanKind, TelemetryProvider
from typing import Any

class LangfuseProvider(TelemetryProvider):
    @property
    def name(self) -> str:
        return "langfuse"

    def start_span(self, kind: SpanKind, name: str, **kwargs) -> str:
        # Create a Langfuse trace/span
        trace = langfuse.trace(name=f"roomkit.{name}")
        return trace.id

    def end_span(self, span_id: str, **kwargs) -> None:
        # End the Langfuse span
        langfuse.span(id=span_id).end(**kwargs)

    def set_attribute(self, span_id: str, key: str, value: Any) -> None:
        langfuse.span(id=span_id).update(metadata={key: value})

    def record_metric(self, name: str, value: float, **kwargs) -> None:
        langfuse.score(name=name, value=value)
```

## Pipeline Metrics

The audio pipeline emits periodic metrics (every 500 frames) instead of per-frame spans to minimize overhead:

- `roomkit.pipeline.frame_count` -- Total frames processed
- `roomkit.pipeline.bytes_processed` -- Total bytes processed

## Provider Metrics

RoomKit records TTFB (time to first byte) and latency metrics inside providers. These are emitted as `record_metric()` calls alongside spans.

### LLM

- `roomkit.llm.ttfb_ms` -- Time from API request to first response (Anthropic, OpenAI, Gemini)

### STT

- `roomkit.stt.ttfb_ms` -- Time from API request to transcription result (Deepgram)
- `roomkit.stt.inference_ms` -- Local inference duration (SherpaOnnx)

### TTS

- `roomkit.tts.ttfb_ms` -- Time from API request to first audio chunk (ElevenLabs)
- `roomkit.tts.synthesis_ms` -- Local synthesis duration (SherpaOnnx)

### Delivery

- `roomkit.delivery.duration_ms` -- End-to-end delivery time in TransportChannel (includes framework overhead)
- `roomkit.delivery.send_ms` -- Provider-level HTTP round-trip time (measured inside each provider's `send()`)

### Realtime

- `roomkit.realtime.uptime_s` -- Session duration from connect to disconnect
- `roomkit.realtime.turn_count` -- Number of completed turns in a session

## Context Manager

The `TelemetryProvider.span()` context manager handles span lifecycle automatically:

```python
with telemetry.span(SpanKind.CUSTOM, "my_operation") as span_id:
    telemetry.set_attribute(span_id, "key", "value")
    # span auto-closes on exit, records error on exception
```
