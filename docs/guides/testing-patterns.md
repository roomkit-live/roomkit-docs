# Testing Patterns

RoomKit provides mock implementations for every pluggable component — AI providers, STT/TTS, voice backends, pipeline stages, memory, identity, and realtime voice. This guide covers how to test RoomKit applications effectively.

## Setup

RoomKit uses pytest with `asyncio_mode = "auto"` — async test functions work without decorators:

```python
from __future__ import annotations

async def test_my_feature() -> None:
    kit = RoomKit()
    # ... test code ...
    assert result.event is not None
```

Run tests with:
```bash
uv run pytest                           # All tests
uv run pytest tests/test_X.py -v        # Specific file
uv run pytest -k "test_broadcast"       # By name pattern
```

---

## Testing Inbound → Broadcast Flow

The most common test pattern — send a message and verify it's broadcast to other channels:

```python
from __future__ import annotations

from roomkit import InboundMessage, RoomKit, TextContent
from roomkit.channels.base import Channel, ChannelCategory, ChannelOutput
from roomkit.models.enums import ChannelType


class SimpleChannel(Channel):
    """Minimal test channel that tracks delivered events."""

    channel_type = ChannelType.SMS

    def __init__(self, channel_id: str) -> None:
        super().__init__(channel_id)
        self.delivered: list = []

    async def handle_inbound(self, message, context):
        from roomkit.models.events import RoomEvent
        return RoomEvent(
            id=f"evt-{message.sender_id}",
            room_id=context.room.id,
            content=message.content,
            source=message.to_event_source(self),
        )

    async def deliver(self, event, binding, context):
        self.delivered.append(event)
        return ChannelOutput.empty()


async def test_message_broadcast() -> None:
    kit = RoomKit()
    ch1 = SimpleChannel("sms1")
    ch2 = SimpleChannel("ws1")
    kit.register_channel(ch1)
    kit.register_channel(ch2)

    await kit.create_room(room_id="r1")
    await kit.attach_channel("r1", "sms1")
    await kit.attach_channel("r1", "ws1")

    msg = InboundMessage(
        channel_id="sms1",
        sender_id="user1",
        content=TextContent(body="hello"),
    )
    result = await kit.process_inbound(msg)

    assert not result.blocked
    assert result.event is not None
    assert len(ch2.delivered) == 1
    assert ch2.delivered[0].content.body == "hello"
```

---

## Mock AI Provider

Test AI-powered channels with deterministic responses:

```python
from __future__ import annotations

from roomkit.providers.ai.mock import MockAIProvider

# Cycle through responses
ai = MockAIProvider(responses=["Hello!", "How can I help?", "Goodbye!"])

# With streaming support
ai = MockAIProvider(responses=["Streamed response"], streaming=True)

# With vision support
ai = MockAIProvider(responses=["I see an image"], vision=True)

# After usage, inspect calls:
assert len(ai.calls) == 3
context = ai.calls[0]  # AIContext from each generate() call
```

### Testing AIChannel

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.channels import AIChannel
from roomkit.providers.ai.mock import MockAIProvider

ai_provider = MockAIProvider(responses=["AI says hello!"])
ai = AIChannel("ai-bot", provider=ai_provider, system_prompt="Be helpful.")

kit = RoomKit()
kit.register_channel(ai)
# ... attach to room, send message, verify AI response ...
```

---

## Mock Voice Components

### MockVoiceBackend

Pure transport mock — no audio processing:

```python
from __future__ import annotations

from roomkit.voice.backends.mock import MockVoiceBackend
from roomkit.voice.pipeline.base import AudioFrame

backend = MockVoiceBackend()

# Create a session
session = await backend.connect("room-1", "user-1", "voice-1")  # low-level backend call
assert session.state.name == "ACTIVE"

# Simulate incoming audio
frame = AudioFrame(data=b"\x00\x01" * 160, sample_rate=16000, channels=1, sample_width=2)
await backend.simulate_audio_received(session, frame)

# Simulate other events
await backend.simulate_barge_in(session)
await backend.simulate_session_ready(session)
await backend.simulate_client_disconnected(session)

# Check what was sent to the client
assert len(backend.sent_audio) > 0
session_id, audio_bytes = backend.sent_audio[0]
```

### MockSTTProvider

```python
from __future__ import annotations

from roomkit.voice.stt.mock import MockSTTProvider

stt = MockSTTProvider(
    transcripts=["Hello", "How are you?", "Goodbye"],
    streaming=False,
)

result = await stt.transcribe(audio_content)
assert result.text == "Hello"

# Cycles through transcripts
result2 = await stt.transcribe(audio_content)
assert result2.text == "How are you?"
```

### MockTTSProvider

```python
from __future__ import annotations

from roomkit.voice.tts.mock import MockTTSProvider

tts = MockTTSProvider(voice="test-voice")

audio = await tts.synthesize("Hello world")
assert audio.url.startswith("https://mock.test/audio/")

# Streaming yields one chunk per word
chunks = []
async for chunk in tts.synthesize_stream("Hello world"):
    chunks.append(chunk)
assert len(chunks) == 2  # "Hello", "world"

# Inspect calls
assert tts.calls[0]["text"] == "Hello world"
assert tts.calls[0]["voice"] == "test-voice"
```

---

## Pipeline Stage Mocks

Every audio pipeline stage has a mock with deterministic event sequences.

### MockVADProvider

```python
from __future__ import annotations

from roomkit.voice.pipeline.vad.base import VADEvent, VADEventType
from roomkit.voice.pipeline.vad.mock import MockVADProvider

# Pre-define the event sequence
vad = MockVADProvider(events=[
    VADEvent(type=VADEventType.SPEECH_START),
    None,                                        # No event (silence frame)
    None,
    VADEvent(type=VADEventType.SPEECH_END, audio_bytes=b"audio-data"),
])

# Each process() call returns the next event
event1 = await vad.process(frame)
assert event1.type == VADEventType.SPEECH_START

event2 = await vad.process(frame)
assert event2 is None  # Silence

# Tracking
assert len(vad.frames) == 2
assert vad.reset_count == 0
```

### MockAECProvider

```python
from __future__ import annotations

from roomkit.voice.pipeline.aec.mock import MockAECProvider

aec = MockAECProvider()

# Process returns frame unchanged (pass-through)
result = await aec.process(frame)
assert result.data == frame.data

# Feed reference audio (for echo cancellation)
await aec.feed_reference(reference_frame)

# Tracking
assert len(aec.frames) == 1
assert len(aec.reference_frames) == 1
```

### Other Pipeline Mocks

All follow the same pattern — event sequence in, deterministic results out:

```python
from __future__ import annotations

from roomkit.voice.pipeline.agc.mock import MockAGCProvider
from roomkit.voice.pipeline.backchannel.mock import MockBackchannelDetector
from roomkit.voice.pipeline.denoiser.mock import MockDenoiserProvider
from roomkit.voice.pipeline.diarization.mock import MockDiarizationProvider
from roomkit.voice.pipeline.dtmf.mock import MockDTMFDetector
from roomkit.voice.pipeline.recorder.mock import MockAudioRecorder
from roomkit.voice.pipeline.resampler.mock import MockResamplerProvider
from roomkit.voice.pipeline.turn.mock import MockTurnDetector

# Diarization with speaker results
diarization = MockDiarizationProvider(results=[
    DiarizationResult(speaker_id="speaker-1", confidence=0.95),
    DiarizationResult(speaker_id="speaker-2", confidence=0.88, is_new_speaker=True),
])

# DTMF with tone events
dtmf = MockDTMFDetector(events=[
    DTMFEvent(digit="1"),
    None,
    DTMFEvent(digit="#"),
])

# Turn detection
turn = MockTurnDetector(decisions=[
    TurnDecision(is_complete=False, confidence=0.3),
    TurnDecision(is_complete=True, confidence=0.95),
])

# Backchannel detection
backchannel = MockBackchannelDetector(decisions=[
    BackchannelDecision(is_backchannel=True, confidence=0.9),
])
```

---

## Voice Pipeline Integration Test

Wire up a full pipeline with mocks to test end-to-end voice flow:

```python
from __future__ import annotations

import asyncio

from roomkit.channels import VoiceChannel
from roomkit.voice.backends.mock import MockVoiceBackend
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.base import AudioFrame
from roomkit.voice.pipeline.vad.base import VADEvent, VADEventType
from roomkit.voice.pipeline.vad.mock import MockVADProvider
from roomkit.voice.stt.mock import MockSTTProvider
from roomkit.voice.tts.mock import MockTTSProvider

# Define pipeline behavior
vad_events = [
    VADEvent(type=VADEventType.SPEECH_START),
    VADEvent(type=VADEventType.SPEECH_END, audio_bytes=b"\x00" * 320),
]

pipeline = AudioPipelineConfig(vad=MockVADProvider(events=vad_events))
backend = MockVoiceBackend()
stt = MockSTTProvider(transcripts=["Hello"])
tts = MockTTSProvider()

channel = VoiceChannel(
    "voice-test",
    backend=backend,
    stt=stt,
    tts=tts,
    pipeline=pipeline,
)

# Start session (low-level backend call for unit tests)
session = await backend.connect("room-1", "user-1", "voice-test")
await backend.simulate_session_ready(session)

# Simulate user speaking
frame = AudioFrame(data=b"\x00\x01" * 160, sample_rate=16000, channels=1, sample_width=2)
await backend.simulate_audio_received(session, frame)
await backend.simulate_audio_received(session, frame)

# Let async tasks complete
await asyncio.sleep(0.2)

# Verify STT received audio
assert len(stt.calls) >= 1

# Verify VAD processed frames
assert len(pipeline.vad.frames) >= 1
```

---

## Mock Memory Provider

Test conversation memory without a real store:

```python
from __future__ import annotations

from roomkit.memory import MockMemoryProvider
from roomkit.providers.ai.base import AIMessage

mock_memory = MockMemoryProvider(
    messages=[AIMessage(role="system", content="Previous conversation summary")],
    events=[event1, event2],
)

# Use with AIChannel
ai = AIChannel("ai", provider=ai_provider, memory=mock_memory)

# After usage, verify calls:
assert len(mock_memory.retrieve_calls) == 1
call = mock_memory.retrieve_calls[0]
assert call.room_id == "room-1"
assert call.channel_id == "ai"

assert len(mock_memory.ingest_calls) >= 1
assert not mock_memory.closed
```

---

## Mock Identity Resolver

Test identity resolution with predefined mappings:

```python
from __future__ import annotations

from roomkit import Identity, MockIdentityResolver
from roomkit.models.enums import IdentificationStatus

alice = Identity(id="alice", display_name="Alice")
bob = Identity(id="bob", display_name="Bob")

resolver = MockIdentityResolver(
    mapping={
        "alice-phone": alice,                    # Known → IDENTIFIED
    },
    ambiguous={
        "shared-phone": [alice, bob],            # Multiple → AMBIGUOUS
    },
    unknown_status=IdentificationStatus.UNKNOWN, # Default for unrecognized
)

kit = RoomKit(identity_resolver=resolver)

# Messages from "alice-phone" → IDENTIFIED as alice
# Messages from "shared-phone" → AMBIGUOUS with [alice, bob]
# Messages from "unknown" → UNKNOWN
```

---

## Mock Realtime Voice Provider

Test speech-to-speech without live API connections:

```python
from __future__ import annotations

from roomkit.channels import RealtimeVoiceChannel
from roomkit.voice.realtime.mock import MockRealtimeProvider, MockRealtimeTransport

provider = MockRealtimeProvider()
transport = MockRealtimeTransport()

channel = RealtimeVoiceChannel("voice-test", provider=provider, transport=transport)

session = await channel.start_session("room-1", "user-1")

# Simulate provider events
await provider.simulate_transcription(session, "Hello", role="user", is_final=True)
await provider.simulate_audio(session, b"\x00\x01" * 100)
await provider.simulate_speech_start(session)
await provider.simulate_speech_end(session)
await provider.simulate_response_start(session)
await provider.simulate_response_end(session)
await provider.simulate_tool_call(session, "call-1", "get_weather", {"city": "NYC"})
await provider.simulate_error(session, "rate_limit", "Too many requests")

# Verify transport interactions
assert len(transport.sent_audio) > 0
assert len(transport.sent_messages) > 0

# Verify provider interactions
assert len(provider.sent_audio) > 0       # Audio forwarded from transport
assert len(provider.tool_results) > 0     # Tool results submitted
```

---

## Hook Testing

### Testing BEFORE_BROADCAST Hooks

```python
from __future__ import annotations

from roomkit import HookResult, HookTrigger, InboundMessage, RoomKit, TextContent


async def test_hook_blocks_message() -> None:
    kit = RoomKit()

    @kit.hook(HookTrigger.BEFORE_BROADCAST)
    async def content_filter(event, ctx):
        if "blocked" in event.content.body:
            return HookResult.block("Content policy violation")
        return HookResult.allow()

    # ... setup channels and room ...

    msg = InboundMessage(
        channel_id="sms1",
        sender_id="user1",
        content=TextContent(body="This is blocked content"),
    )
    result = await kit.process_inbound(msg)

    assert result.blocked
    assert "Content policy" in result.reason
```

### Testing Hook Modification

```python
from __future__ import annotations

from roomkit import HookResult, HookTrigger, RoomKit


async def test_hook_modifies_event() -> None:
    kit = RoomKit()

    @kit.hook(HookTrigger.BEFORE_BROADCAST)
    async def anonymize(event, ctx):
        modified = event.model_copy(
            update={"content": TextContent(body="[redacted]")}
        )
        return HookResult.modify(modified)

    # ... process message ...
    # Delivered content will be "[redacted]"
```

### Testing Hook Injection

```python
from __future__ import annotations

from roomkit import HookResult, HookTrigger, RoomKit
from roomkit.models.events import InjectedEvent, TextContent


async def test_hook_injects_event() -> None:
    kit = RoomKit()

    @kit.hook(HookTrigger.BEFORE_BROADCAST)
    async def auto_reply(event, ctx):
        reply = InjectedEvent(
            content=TextContent(body="Auto-reply: received!"),
            channel_id="sms1",
        )
        return HookResult.allow(injected=[reply])

    # ... process message ...
    # Original message is broadcast, plus the injected auto-reply
```

---

## Mock Telemetry

Track spans and metrics for observability testing:

```python
from __future__ import annotations

from roomkit.telemetry.mock import MockTelemetryProvider

telemetry = MockTelemetryProvider()
kit = RoomKit(telemetry=telemetry)

# ... run operations ...

# Inspect recorded spans
for span in telemetry.completed_spans:
    print(f"{span.kind}: {span.name} ({span.status})")

# Filter by kind
stt_spans = telemetry.get_spans(SpanKind.STT_TRANSCRIBE)

# Check metrics
for metric in telemetry.metrics:
    print(f"{metric['name']}: {metric['value']} {metric['unit']}")
```

---

## Test Utilities

### advance() Helper

Yield control to pending async tasks without real delays:

```python
from __future__ import annotations

import asyncio


async def advance(n: int = 5) -> None:
    """Yield to event loop n times."""
    for _ in range(n):
        await asyncio.sleep(0)


async def test_async_flow() -> None:
    # Start some async work
    task = asyncio.create_task(kit.process_inbound(msg))

    # Let it progress
    await advance()

    # Now check results
    assert task.done()
```

### make_event() Factory

```python
from __future__ import annotations

from roomkit.models.enums import ChannelType
from roomkit.models.events import EventSource, RoomEvent, TextContent


def make_event(
    room_id: str = "test-room",
    channel_id: str = "ch1",
    channel_type: ChannelType = ChannelType.SMS,
    body: str = "test message",
) -> RoomEvent:
    return RoomEvent(
        id=f"evt-{body[:8]}",
        room_id=room_id,
        content=TextContent(body=body),
        source=EventSource(
            channel_id=channel_id,
            channel_type=channel_type,
            participant_id="user-1",
        ),
    )
```

---

## Best Practices

1. **Use mocks for external services** — never call real APIs in unit tests
2. **Test event sequences** — define VAD/STT event sequences that match real scenarios
3. **Check tracking lists** — all mocks record calls for assertion (`mock.calls`, `mock.frames`, etc.)
4. **Use `advance()`** — yield control to async tasks instead of `asyncio.sleep(0.1)`
5. **Test hooks in isolation** — verify block, modify, and inject behaviors independently
6. **Run `make all` before committing** — lint + typecheck + security + tests
