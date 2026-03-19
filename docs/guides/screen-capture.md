# Screen Capture Backend

RoomKit's `ScreenCaptureBackend` captures frames from your screen (or a region of it) and delivers them as standard `VideoFrame` objects. Combined with a `VisionProvider` and `AIChannel`, it enables AI-powered screen assistants that can see, describe, and guide users through software.

## Architecture

```
Screen (mss)
  → ScreenCaptureBackend (background thread, throttled FPS)
    → optional diff-based frame skipping
    → optional downscaling
  → VideoChannel (session lifecycle, hooks, throttled vision sampling)
  → VisionProvider (frame → JPEG → AI API → VisionResult with OCR)
  → setup_video_vision() → AIChannel system prompt
  → AI responds with awareness of what's on screen
```

## Installation

```bash
# Screen capture (always needed)
pip install roomkit[screen-capture]    # mss

# Vision providers (pick one)
pip install roomkit[gemini]            # Gemini (recommended)
pip install roomkit[openai]            # OpenAI GPT-4o / Ollama / vLLM
```

## Quick Start

```python
import asyncio
from roomkit import RoomKit, VideoChannel, AIChannel, ChannelCategory
from roomkit.video.vision.gemini import GeminiVisionConfig, GeminiVisionProvider
from roomkit.providers.ai.mock import MockAIProvider
from roomkit.video.ai_integration import setup_video_vision
from roomkit.video.backends.screen import ScreenCaptureBackend

async def main():
    kit = RoomKit()

    # Screen capture at 2 FPS, half resolution
    backend = ScreenCaptureBackend(monitor=1, fps=2, scale=0.5)

    # Vision AI
    vision = GeminiVisionProvider(GeminiVisionConfig(api_key="AIza..."))

    # Video channel with 5-second analysis interval
    video = VideoChannel("video", backend=backend, vision=vision, vision_interval_ms=5000)
    kit.register_channel(video)

    # AI channel
    ai = AIChannel("ai", provider=MockAIProvider(responses=["I can see your screen!"]))
    kit.register_channel(ai)

    # Wire everything together
    await kit.create_room(room_id="screen-demo")
    await kit.attach_channel("screen-demo", "video")
    await kit.attach_channel("screen-demo", "ai", category=ChannelCategory.INTELLIGENCE)
    setup_video_vision(kit, room_id="screen-demo", ai_channel_id="ai")

    # Start capturing (previously connect_video(), now unified as join())
    session = await kit.join("screen-demo", "video", participant_id="user-1")
    await backend.start_capture(session)

    await asyncio.sleep(60)
    await backend.stop_capture(session)
    await kit.close()

asyncio.run(main())
```

## Constructor Parameters

```python
ScreenCaptureBackend(
    monitor=1,
    region=None,
    fps=5,
    scale=1.0,
    diff_threshold=0.0,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `monitor` | `int` | `1` | Monitor index. `0` = all monitors combined, `1` = primary, `2`+ = secondary. |
| `region` | `tuple[int,int,int,int] \| None` | `None` | Crop region as `(left, top, width, height)` in pixels. Overrides `monitor`. |
| `fps` | `int` | `5` | Capture frame rate. Lower than webcam — screens change less often. |
| `scale` | `float` | `1.0` | Downscale factor in `(0.0, 1.0]`. `0.5` = half resolution. Saves bandwidth and vision API tokens. |
| `diff_threshold` | `float` | `0.0` | Skip frames where pixel change is below this percentage (0.0–1.0). `0.0` = disabled, `0.02` = skip when <2% changed. |

## Features

### Monitor Selection

`mss` enumerates all connected monitors. Index `0` is the union of all displays (virtual desktop), `1` is the primary, and higher indices are additional monitors.

```python
# Primary monitor
backend = ScreenCaptureBackend(monitor=1)

# Secondary monitor
backend = ScreenCaptureBackend(monitor=2)

# All monitors as one image
backend = ScreenCaptureBackend(monitor=0)
```

### Region Cropping

Capture a specific area of the screen instead of the full monitor:

```python
# Capture a 800x600 region starting at (100, 200)
backend = ScreenCaptureBackend(region=(100, 200, 800, 600))
```

When `region` is set, the `monitor` parameter is ignored.

### Downscaling

Reduce frame resolution to save bandwidth and vision API tokens. Scaling uses numpy stride slicing (nearest-neighbor):

```python
# Half resolution — a 1920x1080 screen becomes 960x540
backend = ScreenCaptureBackend(scale=0.5)

# Quarter resolution — good for vision analysis where detail isn't critical
backend = ScreenCaptureBackend(scale=0.25)
```

!!! tip
    For vision analysis, `scale=0.5` is usually sufficient. UI elements and text remain readable, and API costs are reduced ~4x.

### Diff-Based Frame Skipping

Screens are mostly static. Enable diff-based skipping to avoid sending identical frames:

```python
# Skip frames where less than 2% of pixels changed
backend = ScreenCaptureBackend(fps=5, diff_threshold=0.02)
```

The diff check samples a sparse subset of pixels (every 300th byte) for O(1) comparison regardless of resolution. When the frame is too similar to the previous one, it's dropped before reaching the `on_video_received` callback.

| `diff_threshold` | Behavior |
|-------------------|----------|
| `0.0` (default) | All frames delivered |
| `0.01` | Skip if <1% changed (very sensitive) |
| `0.02` | Skip if <2% changed (recommended) |
| `0.05` | Skip if <5% changed (only major changes) |

## Vision Integration

Wire the screen capture to a `VisionProvider` for AI-powered screen understanding:

```python
from roomkit.video.vision.openai import OpenAIVisionConfig, OpenAIVisionProvider

# Ollama with a vision model
vision = OpenAIVisionProvider(OpenAIVisionConfig(
    base_url="http://localhost:11434/v1",
    model="qwen3-vl:8b",
    api_key="ollama",
    prompt=(
        "Describe what software is shown on this screen. "
        "Include visible UI elements, menus, buttons, and text."
    ),
))

video = VideoChannel(
    "video-screen",
    backend=backend,
    vision=vision,
    vision_interval_ms=5000,  # Analyze every 5 seconds
)
```

The `VisionResult` includes a `text` field with OCR output — useful for reading UI labels, error messages, and form fields from the screen.

## Recording Screen Capture

Combine with a video recorder to save screen captures to MP4:

```python
from roomkit.video.pipeline.config import VideoPipelineConfig
from roomkit.video.recorder.pyav import PyAVVideoRecorder
from roomkit.video.recorder import VideoRecordingConfig

recorder = PyAVVideoRecorder()
config = VideoRecordingConfig(storage="./recordings", codec="auto")

video = VideoChannel(
    "video-screen",
    backend=ScreenCaptureBackend(monitor=1, fps=10),
    pipeline=VideoPipelineConfig(recorder=recorder, recording_config=config),
)
```

See the [PyAV Video Recorder guide](pyav-video-recorder.md) for codec options and hardware encoding.

## Hook Triggers

| Trigger | Fires when |
|---------|-----------|
| `ON_VIDEO_SESSION_STARTED` | Screen capture session becomes live |
| `ON_VIDEO_SESSION_ENDED` | Screen capture session disconnects |

```python
@kit.hook(HookTrigger.ON_VIDEO_SESSION_STARTED, execution=HookExecution.ASYNC)
async def on_screen_started(event, ctx):
    print(f"Screen capture started: {event.session.id}")
```

## Capability

`ScreenCaptureBackend` declares `VideoCapability.SCREEN_SHARE`. Use this to distinguish screen capture from webcam sessions:

```python
from roomkit.video.base import VideoCapability

if VideoCapability.SCREEN_SHARE in backend.capabilities:
    print("This is a screen share session")
```

## Backend Comparison

| Backend | Source | Install | Typical FPS | Direction |
|---------|--------|---------|-------------|-----------|
| `ScreenCaptureBackend` | Screen / monitor / region | `roomkit[screen-capture]` | 1–10 | Inbound only |
| `LocalVideoBackend` | Webcam (OpenCV) | `roomkit[local-video]` | 15–30 | Inbound only |
| `RTPVideoBackend` | RTP network stream | `roomkit[rtp]` | 30 | Bidirectional |
| `SIPVideoBackend` | SIP A/V call | `roomkit[sip]` | 30 | Bidirectional |
| `MockVideoBackend` | Simulated frames | Built-in | N/A | Testing |

## Example

Run the screen description example:

```bash
# Mock mode (no API needed)
uv run python examples/screen_describe.py

# Gemini (fast cloud API)
export GEMINI_API_KEY=AIza...
uv run python examples/screen_describe.py --gemini

# Ollama (local)
uv run python examples/screen_describe.py --ollama --model qwen3-vl:8b

# Secondary monitor, quarter resolution
uv run python examples/screen_describe.py --gemini --monitor 2 --scale 0.25

# High frequency with diff skipping
uv run python examples/screen_describe.py --gemini --fps 10 --diff 0.02
```

## Use Cases

| Scenario | Configuration |
|----------|--------------|
| **AI screen assistant** | ScreenCapture + VisionProvider + AIChannel → guided software usage |
| **Screen recording** | ScreenCapture + PyAVVideoRecorder → MP4 screencasts |
| **Accessibility** | ScreenCapture + VisionProvider → AI describes screen for visually impaired users |
| **Remote support** | ScreenCapture + VisionProvider + VoiceChannel → AI sees screen and talks to user |
| **Training & onboarding** | ScreenCapture + VisionProvider → step-by-step software tutorials |
| **Quality assurance** | ScreenCapture + VisionProvider → automated UI state verification |
