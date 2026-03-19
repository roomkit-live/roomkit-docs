# Avatar (Lip-Synced Talking Head)

RoomKit's avatar system generates real-time lip-synced video from TTS audio. When the AI speaks, a portrait photo is animated with matching lip movements and streamed as H.264 video to the caller. When the AI is silent, idle frames (blinking, neutral expression) keep the avatar visible.

## Architecture

```
AI text → TTS → audio chunks ──┬── VoiceBackend (send audio to caller)
                                └── AvatarProvider (audio → video frames)
                                         │
                                    VideoEncoder (RGB → H.264 NALs)
                                         │
                                    VideoBackend (send video RTP)

Idle (no TTS) → AvatarProvider.get_idle_frame() → VideoEncoder → RTP
```

The avatar runs as an **outbound audio tap** on `AudioVideoChannel` — it observes TTS audio without modifying it. Video encoding runs in a thread pool to avoid blocking audio delivery.

## Installation

```bash
# Core (always needed)
pip install roomkit[sip,video]

# For MuseTalk (real lip-sync, requires NVIDIA GPU):
git clone https://github.com/TMElyralab/MuseTalk.git
cd MuseTalk && pip install -r requirements.txt
```

## Quick Start

```python
from __future__ import annotations

import asyncio
from roomkit import AudioVideoChannel, RoomKit, ChannelCategory
from roomkit.channels.ai import AIChannel
from roomkit.video.avatar.mock import MockAvatarProvider
from roomkit.video.backends.sip import SIPVideoBackend
from roomkit.video.pipeline.encoder.pyav import PyAVVideoEncoder

async def main():
    kit = RoomKit()

    # Avatar: mock provider (static image, no GPU needed)
    avatar = MockAvatarProvider(fps=30)
    await avatar.start(open("photo.png", "rb").read(), width=512, height=512)

    # H.264 encoder for video RTP
    encoder = PyAVVideoEncoder(width=512, height=512, fps=30)

    # Combined A/V channel with SIP backend
    backend = SIPVideoBackend(
        local_sip_addr=("0.0.0.0", 5060),
        supported_video_codecs=["H264"],
    )
    av = AudioVideoChannel(
        "voice",
        stt=my_stt,       # your STT provider
        tts=my_tts,        # your TTS provider
        backend=backend,
        avatar=avatar,
        avatar_encoder=encoder,
    )
    kit.register_channel(av)

    # AI channel
    ai = AIChannel("ai", provider=my_ai_provider)
    kit.register_channel(ai)

    # Handle incoming calls
    async def on_call(session):
        await kit.create_room(room_id=session.id)
        await kit.attach_channel(session.id, "voice")
        await kit.attach_channel(session.id, "ai", category=ChannelCategory.INTELLIGENCE)
        # Previously bind_voice_session(), now unified as join() with push model
        await kit.join(session.id, "voice", session=session)

    backend.on_call(on_call)
    await backend.start()
```

## Avatar Providers

### MockAvatarProvider

Displays the reference image as a static frame. No GPU needed — useful for testing the video pipeline.

```python
from roomkit.video.avatar.mock import MockAvatarProvider

avatar = MockAvatarProvider(
    fps=30,
    color=(0, 200, 0),        # frame color during speech
    idle_color=(80, 80, 80),   # frame color when idle (None to disable)
)
await avatar.start(image_bytes, width=512, height=512)
```

### MuseTalkAvatarProvider

Real lip-sync using the [MuseTalk](https://github.com/TMElyralab/MuseTalk) model. Requires NVIDIA GPU with 4GB+ VRAM.

```python
from roomkit.video.avatar.musetalk import MuseTalkAvatarProvider

avatar = MuseTalkAvatarProvider(
    musetalk_dir="/path/to/MuseTalk",   # local git clone
    fps=30,
    batch_size=4,       # higher = smoother, more VRAM
    device="cuda",
    bbox_shift=0,       # face bounding box adjustment
)
await avatar.start(image_bytes, width=512, height=512)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `musetalk_dir` | (required) | Path to MuseTalk git clone |
| `fps` | `30` | Output video frame rate |
| `batch_size` | `4` | Inference batch size |
| `device` | `"cuda"` | PyTorch device |
| `bbox_shift` | `0` | Face crop region shift |

#### MuseTalk setup

```bash
# 1. Clone MuseTalk
git clone https://github.com/TMElyralab/MuseTalk.git

# 2. Install PyTorch with CUDA
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# 3. Install MuseTalk dependencies
cd MuseTalk && pip install -r requirements.txt

# 4. Models download automatically on first run from HuggingFace
```

!!! tip "Running MuseTalk as a separate service"
    MuseTalk's PyTorch/CUDA dependencies are heavy. For production, run it as a
    standalone HTTP service on the GPU machine and create an `HTTPAvatarProvider`
    that calls it — keeping RoomKit lightweight. Community options:

    - **[MuseTalk-API](https://github.com/ruxir-ig/MuseTalk-API)** — Docker + REST wrapper (batch mode, not real-time)
    - **[fal.ai](https://fal.ai/models/fal-ai/musetalk)** — hosted cloud API (batch mode)
    - **Custom FastAPI service** — wrap `feed_audio()` / `get_idle_frame()` in a streaming HTTP endpoint for real-time use

### Custom AvatarProvider

Implement the `AvatarProvider` ABC for any lip-sync model:

```python
from roomkit.video.avatar.base import AvatarProvider
from roomkit.video.video_frame import VideoFrame

class MyAvatarProvider(AvatarProvider):
    @property
    def name(self) -> str:
        return "my-avatar"

    @property
    def fps(self) -> int:
        return 30

    @property
    def is_started(self) -> bool:
        return self._started

    async def start(self, reference_image: bytes, *, width: int = 512, height: int = 512) -> None:
        """Load model and preprocess reference face."""
        self._started = True

    def feed_audio(self, pcm_data: bytes, sample_rate: int = 16000) -> list[VideoFrame]:
        """Generate lip-synced frames from audio. Called per TTS chunk."""
        return [VideoFrame(data=rgb_bytes, codec="raw_rgb24", width=512, height=512)]

    def get_idle_frame(self) -> VideoFrame | None:
        """Return idle animation frame (blinking, neutral). Called at fps rate."""
        return None

    def flush(self) -> list[VideoFrame]:
        """Return remaining frames after speech ends."""
        return []

    async def stop(self) -> None:
        self._started = False
```

## Video Size

The avatar resolution is set in three places that must match:

```python
# 1. Avatar provider — resizes the reference image
await avatar.start(image_bytes, width=640, height=480)

# 2. H.264 encoder — output NAL dimensions
encoder = PyAVVideoEncoder(width=640, height=480, fps=30)
```

The example supports `--size`:

```bash
python examples/avatar_call.py --image photo.png --size 640x480
python examples/avatar_call.py --image photo.png --size 1280x720
```

## Idle Frame Loop

When TTS is not playing, `AudioVideoChannel` runs a background loop that calls `avatar.get_idle_frame()` at the configured fps rate. This keeps the avatar visible between speech — showing the reference image, blinking animation, or neutral expression.

The loop automatically pauses during TTS playback (the audio tap generates lip-synced frames instead) and resumes when TTS finishes.

## How It Works

1. **Session binds** → idle frame loop starts, caller sees the avatar immediately
2. **AI responds** → TTS generates audio chunks
3. **Outbound audio tap** fires for each chunk → `avatar.feed_audio()` produces `VideoFrame`s
4. **Thread pool** encodes each frame to H.264 (avoids blocking audio delivery)
5. **SIP video RTP** sends encoded NALs to the caller
6. **TTS ends** → idle frame loop resumes

```
kit.join(room_id, channel_id, session=session)
  ├─ Idle loop: get_idle_frame() → encode → RTP (at fps rate)
  │
  ├─ TTS playing: feed_audio() → encode → RTP (driven by audio chunks)
  │                idle loop paused automatically
  │
  └─ TTS ends: idle loop resumes
      │
kit.leave(session)
  └─ Idle loop cancelled
```

## Echo Cancellation

When avatar audio plays through the caller's speaker and feeds back into their microphone, AEC (Acoustic Echo Cancellation) prevents false barge-in interruptions:

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.aec.webrtc import WebRTCAECProvider

av = AudioVideoChannel(
    "voice",
    backend=backend,
    stt=stt,
    tts=tts,
    avatar=avatar,
    avatar_encoder=encoder,
    pipeline=AudioPipelineConfig(aec=WebRTCAECProvider(sample_rate=16000)),
)
```

## Full Example

See [`examples/avatar_call.py`](https://github.com/anthropics/roomkit/blob/main/examples/avatar_call.py) for a complete SIP avatar agent with Deepgram STT, ElevenLabs TTS, Claude AI, and room recording.

```bash
# Mock avatar (no GPU)
uv run python examples/avatar_call.py --image photo.png --size 512x512

# MuseTalk (GPU required)
uv run python examples/avatar_call.py --image photo.png --musetalk ~/MuseTalk --size 300x300
```
