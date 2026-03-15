# Anam AI Avatar (Realtime Audio+Video)

RoomKit's `RealtimeAudioVideoChannel` connects to [Anam AI](https://anam.ai) to deliver photorealistic talking-head avatars in real-time. Anam handles STT, LLM reasoning, TTS, and face animation in the cloud — delivering synchronized audio and video over WebRTC.

## Architecture

```
User audio → Transport (SIP/WS) → RealtimeAudioVideoChannel
                                         |
                                   AnamRealtimeProvider
                                         |  (WebRTC via anam SDK)
                                     Anam Cloud
                                  STT → LLM → TTS → Avatar
                                         |
                                audio + video frames
                                         |
                                on_audio() → transport → user
                                on_video() → video taps / vision
```

Unlike `VoiceChannel` (STT → AI → TTS) or `RealtimeVoiceChannel` (audio-only speech-to-speech), `RealtimeAudioVideoChannel` adds a video output modality. The provider delivers both audio and video frames, which the channel routes to taps, vision analysis, and recording.

---

## Installation

```bash
pip install roomkit[anam]
# or with SIP transport:
pip install roomkit[anam,sip,video]
```

---

## Quick Start

```python
from roomkit import (
    AnamConfig,
    AnamRealtimeProvider,
    RealtimeAudioVideoChannel,
    RoomKit,
)
from roomkit.voice.realtime.mock import MockRealtimeTransport

# Configure Anam (ephemeral persona)
config = AnamConfig(
    api_key="your-api-key",
    avatar_id="your-avatar-id",
    voice_id="your-voice-id",
    llm_id="your-llm-id",
    system_prompt="You are a helpful AI avatar.",
)
provider = AnamRealtimeProvider(config)

transport = MockRealtimeTransport()

channel = RealtimeAudioVideoChannel(
    "avatar",
    provider=provider,
    transport=transport,
)

kit = RoomKit()
kit.register_channel(channel)
```

---

## AnamConfig

Two persona modes are supported:

### Pre-defined Persona (from Anam Lab)

```python
config = AnamConfig(
    api_key="your-api-key",
    persona_id="your-persona-id",
)
```

### Ephemeral Persona (full control)

Configure individual components from [lab.anam.ai](https://lab.anam.ai):

```python
config = AnamConfig(
    api_key="your-api-key",
    avatar_id="30fa96d0-...",         # from lab.anam.ai/avatars
    avatar_model="cara-3",            # optional model variant
    voice_id="6bfbe25a-...",          # from lab.anam.ai/voices
    llm_id="0934d97d-...",            # from lab.anam.ai/llms
    system_prompt="You are a concise assistant.",
    language_code="en",
    enable_audio_passthrough=False,    # False = Anam manages TTS context
    timeout=30.0,
)
```

| Parameter | Description |
|-----------|-------------|
| `api_key` | Anam API key (required) |
| `persona_id` | Pre-defined persona from Anam Lab |
| `avatar_id` | Avatar face model ID |
| `avatar_model` | Video frame model (e.g. `"cara-3"`) |
| `voice_id` | TTS voice ID |
| `llm_id` | LLM model ID |
| `system_prompt` | System instructions for the LLM |
| `language_code` | BCP-47 language code (default: `"en"`) |
| `enable_audio_passthrough` | Bypass Anam's TTS context tracking |
| `timeout` | Connection timeout in seconds (default: `30.0`) |

---

## RealtimeAudioVideoChannel

Extends `RealtimeVoiceChannel` with video capabilities:

```python
channel = RealtimeAudioVideoChannel(
    "avatar",
    provider=provider,                # RealtimeAudioVideoProvider
    transport=transport,              # VoiceBackend (audio transport)
    video_pipeline=None,              # Optional VideoPipelineConfig
    vision=None,                      # Optional VisionProvider
    vision_interval_ms=2000,          # Vision analysis throttle
    system_prompt="...",              # Forwarded to provider
    voice="...",                      # Forwarded to provider
)
```

### Video Taps

Register callbacks that receive every video frame (for recording, display, or custom processing):

```python
def on_video(session, frame):
    print(f"Frame: {frame.width}x{frame.height}, codec={frame.codec}")

channel.add_video_media_tap(on_video)
```

### Vision Analysis

Attach a vision provider for periodic frame analysis:

```python
from roomkit import OpenAIVisionProvider, OpenAIVisionConfig

channel = RealtimeAudioVideoChannel(
    "avatar",
    provider=provider,
    transport=transport,
    vision=OpenAIVisionProvider(OpenAIVisionConfig(api_key="sk-...")),
    vision_interval_ms=3000,  # Analyze every 3 seconds
)
```

---

## Session Lifecycle

```python
# Start a session (connects transport + Anam provider)
session = await channel.start_session(
    room_id="room-1",
    participant_id="user-1",
    connection=websocket,   # or SIP session, or mock
)

# Session is now active — Anam streams audio+video
# Audio goes to transport → user
# Video goes to registered taps

# End the session
await channel.end_session(session)
```

### Hooks

Video session hooks fire automatically:

```python
@kit.hook(HookTrigger.ON_VIDEO_SESSION_STARTED)
async def on_video_started(event, ctx):
    print(f"Avatar video started: {event.session.id}")

@kit.hook(HookTrigger.ON_VIDEO_SESSION_ENDED)
async def on_video_ended(event, ctx):
    print(f"Avatar video ended: {event.session.id}")
```

---

## SIP Integration

Bridge SIP phone calls to an Anam avatar — the caller talks to a photorealistic AI over a standard video call:

```python
from roomkit import AnamConfig, AnamRealtimeProvider, RealtimeAudioVideoChannel, RoomKit
from roomkit.video.backends.sip import SIPVideoBackend
from roomkit.voice.realtime.mock import MockRealtimeTransport

# SIP backend for caller transport
sip = SIPVideoBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    supported_video_codecs=["H264"],
)

# Anam provider
provider = AnamRealtimeProvider(AnamConfig(
    api_key="...",
    avatar_id="...",
    voice_id="...",
    llm_id="...",
))

channel = RealtimeAudioVideoChannel(
    "anam-sip",
    provider=provider,
    transport=MockRealtimeTransport(),
)

kit = RoomKit()
kit.register_channel(channel)

# Route incoming calls
async def on_call(session):
    room_id = session.metadata["room_id"]
    await kit.create_room(room_id=room_id)
    await kit.attach_channel(room_id, "anam-sip")
    await channel.start_session(room_id, session.metadata["caller"], session)

sip.on_call(on_call)
await sip.start()
```

See `examples/sip_anam_avatar.py` for a complete runnable example.

---

## Provider API

`AnamRealtimeProvider` wraps the [Anam Python SDK](https://docs.anam.ai/sdk-reference/python-sdk):

| Method | Description |
|--------|-------------|
| `send_audio(session, bytes)` | Forward raw PCM audio to Anam (`send_user_audio`) |
| `inject_text(session, text)` | Send text through the LLM (`send_message`) |
| `interrupt(session)` | Cancel current avatar response |
| `on_audio(callback)` | Register audio output callback |
| `on_video(callback)` | Register video frame callback |
| `on_transcription(callback)` | Register transcription callback |

### Frame Format

Video frames from Anam arrive as `VideoFrame(codec="raw_rgb24")` — raw RGB pixels converted from PyAV. Audio arrives as PCM int16 bytes.

---

## Testing

Use `MockRealtimeAudioVideoProvider` for testing without Anam credentials:

```python
from roomkit.voice.realtime.mock import (
    MockRealtimeAudioVideoProvider,
    MockRealtimeTransport,
)
from roomkit.video.video_frame import VideoFrame

provider = MockRealtimeAudioVideoProvider()
transport = MockRealtimeTransport()

channel = RealtimeAudioVideoChannel(
    "test-avatar",
    provider=provider,
    transport=transport,
)

# Simulate video from the provider
frame = VideoFrame(
    data=b"\x00" * (640 * 480 * 3),
    codec="raw_rgb24",
    width=640,
    height=480,
)
await provider.simulate_video(session, frame)
```

---

## Examples

| Example | Description |
|---------|-------------|
| `examples/realtime_av_anam.py` | Basic Anam avatar with mock transport |
| `examples/sip_anam_avatar.py` | SIP-to-Anam bridge (phone → avatar) |
