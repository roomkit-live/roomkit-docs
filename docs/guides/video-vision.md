# Video & Vision

RoomKit's video subsystem adds real-time video capture and AI-powered frame analysis to multi-channel rooms. A camera feed can be analyzed by vision AI (Gemini, Ollama, OpenAI) and the results injected into AI conversation context — enabling agents that can "see".

## Architecture

```
Camera (LocalVideoBackend)
  → VideoChannel (session lifecycle, hooks, throttled sampling)
  → VisionProvider (frame → JPEG → AI API → VisionResult)
  → setup_video_vision() → AIChannel system prompt
  → AI responds with awareness of the visual scene
```

## Installation

```bash
# Core (always needed)
pip install roomkit

# Local webcam capture
pip install roomkit[local-video]    # opencv-python-headless

# Vision providers (pick one)
pip install roomkit[gemini]         # Gemini 2.5 Flash (recommended)
pip install roomkit[openai]         # OpenAI GPT-4o / Ollama / vLLM
```

## Quick Start

```python
import asyncio
from roomkit import (
    RoomKit, VideoChannel, AIChannel, ChannelCategory,
    GeminiVisionConfig, GeminiVisionProvider,
    MockAIProvider, setup_video_vision,
)
from roomkit.video.backends.local import LocalVideoBackend

async def main():
    kit = RoomKit()

    # Video: webcam + Gemini vision
    backend = LocalVideoBackend(device=0, fps=15)
    vision = GeminiVisionProvider(GeminiVisionConfig(api_key="AIza..."))
    video = VideoChannel("video", backend=backend, vision=vision)
    kit.register_channel(video)

    # AI agent
    ai = AIChannel("ai", provider=MockAIProvider(responses=["I see!"]))
    kit.register_channel(ai)

    # Room: wire video → AI
    await kit.create_room(room_id="demo")
    await kit.attach_channel("demo", "video")
    await kit.attach_channel("demo", "ai", category=ChannelCategory.INTELLIGENCE)
    setup_video_vision(kit, room_id="demo", ai_channel_id="ai")

    # Start capturing (previously connect_video(), now unified as join())
    session = await kit.join("demo", "video", participant_id="user-1")
    await backend.start_capture(session)

    await asyncio.sleep(30)  # Run for 30 seconds
    await backend.stop_capture(session)
    await kit.close()

asyncio.run(main())
```

## Components

### VideoBackend

Transport abstraction for video sources. Two implementations:

| Backend | Use case | Install |
|---------|----------|---------|
| `LocalVideoBackend` | Webcam capture for dev/testing | `roomkit[local-video]` |
| `ScreenCaptureBackend` | Screen/monitor capture | `roomkit[screen-capture]` |
| `MockVideoBackend` | Unit tests | Built-in |

```python
from roomkit.video.backends.local import LocalVideoBackend

backend = LocalVideoBackend(
    device=0,       # Camera index
    fps=15,         # Capture rate
    width=640,      # Resolution
    height=480,
)
```

### VisionProvider

Analyzes video frames and returns structured descriptions. Three implementations:

| Provider | Backend | Install |
|----------|---------|---------|
| `GeminiVisionProvider` | Google Gemini 2.5 Flash | `roomkit[gemini]` |
| `OpenAIVisionProvider` | OpenAI, Ollama, vLLM | `roomkit[openai]` |
| `MockVisionProvider` | Unit tests | Built-in |

#### Gemini (recommended for speed)

```python
from roomkit import GeminiVisionConfig, GeminiVisionProvider

vision = GeminiVisionProvider(GeminiVisionConfig(
    api_key="AIza...",
    model="gemini-2.5-flash",
))
```

#### Ollama (local, free)

```python
from roomkit import OpenAIVisionConfig, OpenAIVisionProvider

vision = OpenAIVisionProvider(OpenAIVisionConfig(
    base_url="http://localhost:11434/v1",
    model="qwen3-vl:8b",
    api_key="ollama",
))
```

### VideoChannel

Session-based channel that wires a `VideoBackend` to a room:

- Manages session lifecycle (connect, bind, disconnect)
- Fires `ON_VIDEO_SESSION_STARTED` / `ON_VIDEO_SESSION_ENDED` hooks
- Runs `VisionProvider` at configurable intervals
- Emits `video_vision_result` framework events

```python
video = VideoChannel(
    "video-main",
    backend=backend,
    vision=vision,
    vision_interval_ms=3000,  # Analyze every 3 seconds
)
```

### AI Integration

`setup_video_vision()` wires vision results into an AIChannel's system prompt:

```python
from roomkit import setup_video_vision

setup_video_vision(
    kit,
    room_id="my-room",
    ai_channel_id="ai",
    context_prefix="You can see a live camera feed. Current view:",
)
```

The AI's system prompt is automatically updated with the latest vision description every time a frame is analyzed. The base system prompt is preserved.

## Video Recording

Two recorder implementations are available:

| Recorder | Codec | Compression | Install |
|----------|-------|-------------|---------|
| `PyAVVideoRecorder` | H.264 / H.265 / NVENC | High (10-50x smaller) | `roomkit[video]` |
| `OpenCVVideoRecorder` | mp4v (MPEG-4 Part 2) | Low | `roomkit[local-video]` |
| `MockVideoRecorder` | None | In-memory | Built-in |

**PyAV (recommended for production)** — compressed H.264 MP4 via FFmpeg:

```python
from roomkit import VideoPipelineConfig
from roomkit.video.recorder.pyav import PyAVVideoRecorder
from roomkit.video.recorder import VideoRecordingConfig

recorder = PyAVVideoRecorder()
config = VideoRecordingConfig(storage="./recordings", codec="auto")

video = VideoChannel(
    "video",
    backend=backend,
    pipeline=VideoPipelineConfig(recorder=recorder, recording_config=config),
)
```

**OpenCV** — quick path, larger files:

```python
from roomkit import VideoPipelineConfig
from roomkit.video.recorder.opencv import OpenCVVideoRecorder
from roomkit.video.recorder import VideoRecordingConfig

recorder = OpenCVVideoRecorder()
config = VideoRecordingConfig(storage="./recordings")

video = VideoChannel(
    "video",
    backend=backend,
    pipeline=VideoPipelineConfig(recorder=recorder, recording_config=config),
)
```

Recording starts automatically when a session binds and stops when it unbinds. Every received frame is tapped to the recorder and the file is finalized on stop.

See the [PyAV Video Recorder guide](pyav-video-recorder.md) for codec selection, NVENC hardware encoding, and compression details.

```bash
# PyAV recorder (H.264)
uv run python examples/pyav_video_recorder.py
uv run python examples/pyav_video_recorder.py --codec h264_nvenc --duration 30

# OpenCV recorder (raw)
uv run python examples/webcam_recording.py
```

## Hook Triggers

| Trigger | Fires when |
|---------|-----------|
| `ON_VIDEO_SESSION_STARTED` | Video session becomes live |
| `ON_VIDEO_SESSION_ENDED` | Video session disconnects |
| `ON_VIDEO_TRACK_ADDED` | New video track added (future) |
| `ON_VIDEO_TRACK_REMOVED` | Video track removed (future) |
| `ON_SCREEN_SHARE_STARTED` | Screen share begins (future) |
| `ON_SCREEN_SHARE_STOPPED` | Screen share ends (future) |

```python
@kit.hook(HookTrigger.ON_VIDEO_SESSION_STARTED, HookExecution.ASYNC)
async def on_video_start(event, ctx):
    print(f"Video session started: {event.session.id}")
```

## Example

Run the webcam vision example:

```bash
# Mock mode (no API needed)
uv run python examples/webcam_vision.py

# Gemini (fast cloud API)
export GEMINI_API_KEY=AIza...
uv run python examples/webcam_vision.py --gemini

# Ollama (local)
uv run python examples/webcam_vision.py --ollama --model qwen3-vl:8b

# French output
uv run python examples/webcam_vision.py --gemini --lang fr
```

## Use Cases

| Scenario | Configuration |
|----------|--------------|
| **Security camera** | VideoChannel + VisionProvider → text alerts |
| **SIP video interphone** | VideoChannel + VoiceChannel → AI sees and talks |
| **Quality inspection** | VideoChannel + VisionProvider → defect detection |
| **Accessibility** | VideoChannel → AI describes visual scene for blind users |
| **Screen assistant** | [ScreenCaptureBackend](screen-capture.md) + VisionProvider → AI-guided software usage |
