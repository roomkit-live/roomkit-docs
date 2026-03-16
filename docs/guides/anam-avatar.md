# Anam AI Avatar (Realtime Audio+Video)

RoomKit integrates with [Anam AI](https://anam.ai) to deliver photorealistic talking-head avatars in real-time. Anam handles STT, LLM reasoning, TTS, and face animation in the cloud — delivering synchronized audio and video over WebRTC.

## Architecture

Two integration patterns are available:

### RealtimeAudioVideoChannel (room-based)

For applications that need RoomKit's full room model (hooks, multi-channel, events):

```
User audio → Transport (WS) → RealtimeAudioVideoChannel → hooks/events
                                       ↕
                                AnamRealtimeProvider
                                       ↕  (WebRTC)
                                   Anam Cloud
                                STT → LLM → TTS → Avatar
                                       ↕
                              audio + video frames
```

### RealtimeAVBridge (direct backend wiring)

For bridging any voice/video backend (SIP, RTP, local mic) directly to Anam — no room model needed:

```
SIP phone → RTP audio → RealtimeAVBridge → provider.send_audio() → Anam STT
                              ↕
                       AnamRealtimeProvider (WebRTC)
                              ↕
                         Anam Cloud
                    STT → LLM → TTS → Avatar
                              ↕
              audio → resample → SIP RTP → phone speaker
              video → pipeline → H.264 encode → SIP RTP → phone screen
```

The bridge handles audio resampling (48kHz → SIP codec rate), video encoding (raw RGB → H.264), session lifecycle, and graceful cleanup.

---

## Installation

```bash
pip install roomkit[anam]
# With SIP transport + video encoding:
pip install roomkit[anam,sip,video]
```

---

## Quick Start

```python
from roomkit import AnamConfig, AnamRealtimeProvider
from roomkit.voice.realtime.bridge import RealtimeAVBridge
from roomkit.video.backends.sip import SIPVideoBackend
from roomkit.video.pipeline.encoder.pyav import PyAVVideoEncoder

sip = SIPVideoBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="0.0.0.0",
    supported_video_codecs=["H264"],
)

provider = AnamRealtimeProvider(AnamConfig(
    api_key="your-api-key",
    avatar_id="your-avatar-id",
    voice_id="your-voice-id",
    llm_id="your-llm-id",
))

bridge = RealtimeAVBridge(
    provider, sip,
    encoder=PyAVVideoEncoder(fps=25),
)

await sip.start()
# Incoming SIP calls are now connected to Anam automatically.
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
    language_code="fr",               # BCP-47 language code
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

## RealtimeAVBridge

Generic bridge that wires any `VoiceBackend` to any `RealtimeAudioVideoProvider`. Handles:

- **Audio in**: Backend → `provider.send_audio()` (user speech → AI)
- **Audio out**: Provider → resample (48kHz → codec rate) → `backend.send_audio()` (AI speech → user)
- **Video out**: Provider → pipeline → encode → backend RTP (avatar → user)
- **Session lifecycle**: Auto-connect on SIP INVITE, auto-disconnect on BYE

```python
from roomkit.voice.realtime.bridge import RealtimeAVBridge
from roomkit.video.pipeline.encoder.pyav import PyAVVideoEncoder
from roomkit.video.utils import make_text_frame

bridge = RealtimeAVBridge(
    provider,                            # Any RealtimeAudioVideoProvider
    backend,                             # Any VoiceBackend (SIP, RTP, local)
    encoder=PyAVVideoEncoder(fps=25),    # None = passthrough (taps only)
    video_pipeline=pipeline_config,      # Optional VideoPipelineConfig
    connecting_frame=make_text_frame(    # Shown during provider negotiation
        "Connecting to avatar...\nPlease wait",
    ),
    provider_sample_rate=48000,          # Anam outputs 48kHz
    on_transcription=my_handler,         # Optional callbacks
)
```

### Connecting Placeholder

Anam's WebRTC negotiation takes 3-5 seconds. During this time, the caller sees a black screen by default. Use `connecting_frame` to show a placeholder instead:

```python
from roomkit.video.utils import make_text_frame

bridge = RealtimeAVBridge(
    provider, sip,
    encoder=PyAVVideoEncoder(fps=25),
    connecting_frame=make_text_frame("Connecting to avatar...\nPlease wait"),
    connecting_fps=5,  # Low fps is fine for a static image
)
```

`make_text_frame()` generates a `VideoFrame` with centered multi-line text on a dark background. The placeholder is sent at `connecting_fps` (default 5) until the provider connects, then real avatar frames take over automatically.

You can customize the frame appearance:

```python
make_text_frame(
    "Bienvenue\nConnexion en cours...",
    width=720, height=480,
    bg_color=(20, 20, 50),         # Dark blue background
    text_color=(255, 255, 255),    # White title
    sub_color=(180, 180, 200),     # Light grey subtitle
    font_scale=1.0,
)
```

### Video Pipeline

Insert processing stages between provider frames and the encoder:

```python
from roomkit import VideoPipelineConfig
from roomkit.video.pipeline.filter.watermark import WatermarkFilter

bridge = RealtimeAVBridge(
    provider, sip,
    video_pipeline=VideoPipelineConfig(
        filters=[
            WatermarkFilter(
                "RoomKit | {timestamp}",
                position="bottom-left",
                color=(255, 255, 255),
                bg_color=(0, 0, 0),
            ),
        ],
    ),
    encoder=PyAVVideoEncoder(fps=25, bitrate=3_000_000),
)
```

The pipeline supports all standard stages: resizer, filters (watermark, YOLO, censor), transforms, and vision analysis.

### Video Encoder

The encoder converts raw RGB frames to H.264 for SIP/RTP:

```python
from roomkit.video.pipeline.encoder.pyav import PyAVVideoEncoder

# Default: 2.5 Mbps, veryfast preset
encoder = PyAVVideoEncoder(fps=25)

# Higher quality for LAN
encoder = PyAVVideoEncoder(fps=25, bitrate=3_000_000, preset="medium")
```

Without an encoder, video goes to taps only (passthrough mode for local display).

### Video Taps

Register callbacks that receive every video frame:

```python
bridge.add_video_tap(lambda session, frame: print(f"{frame.width}x{frame.height}"))
```

### Manual Connect (local mic/webcam)

For backends that don't fire `on_call` automatically:

```python
session = await bridge.connect(backend_session, participant_id="user-1")
# ...
await bridge.disconnect(session.id)
```

---

## SIP Integration

Bridge SIP phone calls to Anam — the caller talks to a photorealistic AI avatar:

```python
from roomkit import AnamConfig, AnamRealtimeProvider, VideoPipelineConfig
from roomkit.video.backends.sip import SIPVideoBackend
from roomkit.video.pipeline.encoder.pyav import PyAVVideoEncoder
from roomkit.video.pipeline.filter.watermark import WatermarkFilter
from roomkit.video.utils import make_text_frame
from roomkit.voice.realtime.bridge import RealtimeAVBridge

sip = SIPVideoBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="0.0.0.0",
    supported_video_codecs=["H264"],
)

provider = AnamRealtimeProvider(AnamConfig(
    api_key="...",
    avatar_id="...",
    voice_id="...",
    llm_id="...",
    language_code="fr",
    system_prompt="Tu es un avatar IA serviable.",
))

bridge = RealtimeAVBridge(
    provider, sip,
    video_pipeline=VideoPipelineConfig(
        filters=[WatermarkFilter("RoomKit | {timestamp}")],
    ),
    connecting_frame=make_text_frame("Connexion en cours...\nVeuillez patienter"),
    encoder=PyAVVideoEncoder(fps=25, bitrate=3_000_000),
    on_transcription=lambda role, text, _: print(f"[{role}] {text}"),
)

await sip.start()
```

See `examples/sip_anam_avatar.py` for a complete runnable example with signal handling and graceful shutdown.

### Known Characteristics

- **Startup delay (~4-5s)**: Anam's WebRTC negotiation (ICE gathering, TURN allocation). The SIP phone answers immediately but the avatar needs time to connect. This is inherent to the Anam SDK.
- **Video re-encoding**: The Anam SDK decodes WebRTC video to raw pixels before exposing frames. RoomKit re-encodes to H.264 for SIP. This is a limitation of the SDK (no raw H.264 access).
- **Audio resampling**: Anam outputs 48kHz stereo PCM (WebRTC OPUS decoded). The bridge resamples to the SIP codec rate (8kHz G.711, 16kHz G.722) automatically.

---

## RealtimeAudioVideoChannel (Room-Based)

For applications that need RoomKit's room model:

```python
from roomkit import (
    AnamConfig, AnamRealtimeProvider,
    RealtimeAudioVideoChannel, RoomKit,
)
from roomkit.voice.realtime.mock import MockRealtimeTransport

provider = AnamRealtimeProvider(AnamConfig(
    api_key="...", avatar_id="...", voice_id="...", llm_id="...",
))

channel = RealtimeAudioVideoChannel(
    "avatar",
    provider=provider,
    transport=MockRealtimeTransport(),
    vision=my_vision_provider,        # Optional
    vision_interval_ms=3000,
)

kit = RoomKit()
kit.register_channel(channel)

session = await channel.start_session("room-1", "user-1", connection)
```

### Hooks

```python
@kit.hook(HookTrigger.ON_VIDEO_SESSION_STARTED)
async def on_video_started(event, ctx):
    print(f"Avatar video started: {event.session.id}")

@kit.hook(HookTrigger.ON_VIDEO_SESSION_ENDED)
async def on_video_ended(event, ctx):
    print(f"Avatar video ended: {event.session.id}")
```

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

- **Video**: `VideoFrame(codec="raw_rgb24")` — raw RGB pixels (720x480 typical)
- **Audio**: PCM int16 mono at 48kHz (downmixed from stereo by the provider)

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

frame = VideoFrame(
    data=b"\x00" * (640 * 480 * 3),
    codec="raw_rgb24",
    width=640, height=480,
)
await provider.simulate_video(session, frame)
```

---

## Examples

| Example | Description |
|---------|-------------|
| `examples/realtime_av_anam.py` | Basic Anam avatar with mock transport |
| `examples/sip_anam_avatar.py` | SIP-to-Anam bridge with video pipeline and H.264 encoding |
