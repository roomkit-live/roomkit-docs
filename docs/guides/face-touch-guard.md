# Face Touch Guard

Detect hand-to-face contact in real-time video using MediaPipe landmarks. The `FaceTouchFilter` runs in the video pipeline and fires `ON_VIDEO_DETECTION` hooks when a confirmed touch is detected, enabling audio alerts, messages, or any custom reaction.

Inspired by [FaceTouchGuard](https://github.com/timpratim/faceguard) — a local desktop app that alerts you when you touch your face.

## Architecture

```
Camera (VideoBackend)
  → VideoChannel
  → VideoPipeline.process_inbound()
  → FaceTouchFilter (MediaPipe Face Landmarker + Hand Landmarker)
    → Zone geometry: fingertip distance to face zone centroids
    → False-positive filtering: proximity, z-depth, confirmation, cooldown
    → FilterEvent(kind="face_touch") → context.events
  → VideoChannel drains events → ON_VIDEO_DETECTION hook
  → User hook handler (log, alert, TTS, message, etc.)
```

## Installation

```bash
pip install roomkit[mediapipe]

# For local webcam capture
pip install roomkit[local-video,mediapipe]
```

## Quick Start

```python
import asyncio
from roomkit import RoomKit, HookTrigger, HookExecution, VideoDetectionEvent
from roomkit.channels.video import VideoChannel
from roomkit.video.backends.local import LocalVideoBackend
from roomkit.video.pipeline.config import VideoPipelineConfig
from roomkit.video.pipeline.filter.mediapipe_face_touch import (
    FaceTouchConfig,
    FaceTouchFilter,
    FaceTouchSensitivity,
)

async def main():
    kit = RoomKit()

    backend = LocalVideoBackend(device=0, fps=15)
    pipeline = VideoPipelineConfig(
        filters=[FaceTouchFilter(FaceTouchConfig(
            sensitivity=FaceTouchSensitivity.HIGH,
        ))],
    )
    video = VideoChannel("video-cam", backend=backend, pipeline=pipeline)
    kit.register_channel(video)

    @kit.hook(HookTrigger.ON_VIDEO_DETECTION, execution=HookExecution.ASYNC)
    async def on_touch(event: VideoDetectionEvent, ctx):
        if event.kind == "face_touch":
            zone = event.metadata.get("zone")
            print(f"Stop touching your {zone}!")

    room = await kit.create_room("guard-room")
    await kit.bind_channel(room.id, "video-cam")

asyncio.run(main())
```

## Face Zones

The filter monitors five face zones defined by MediaPipe's 478-landmark face mesh:

| Zone | Description | Default |
|------|-------------|---------|
| `left_cheek` | Eye-cheek boundary to jawline (left) | Enabled |
| `right_cheek` | Mirror on right side | Enabled |
| `chin` | Lower jaw contour | Enabled |
| `mouth` | Outer lip perimeter | Enabled |
| `forehead` | Eyebrows to hairline | Disabled (higher FP rate) |

Select zones via the `zones` parameter:

```python
from roomkit.video.pipeline.filter.mediapipe_face_touch import FaceZone

config = FaceTouchConfig(
    zones=frozenset({FaceZone.LEFT_CHEEK, FaceZone.RIGHT_CHEEK, FaceZone.FOREHEAD}),
)
```

## Sensitivity Presets

Three presets control detection thresholds:

| Preset | Distance | Confirmation | Cooldown | Z-depth |
|--------|----------|-------------|----------|---------|
| `LOW` | 0.04 | 4 frames | 30 frames | 0.08 |
| `MEDIUM` | 0.06 | 3 frames | 20 frames | 0.08 |
| `HIGH` | 0.08 | 2 frames | 12 frames | 0.12 |

- **Distance threshold** — normalized 2D distance from fingertip to zone centroid (resolution-independent)
- **Confirmation frames** — consecutive positive frames required before triggering
- **Cooldown frames** — suppress re-firing for the same zone after a detection
- **Z-depth threshold** — reject hands hovering in front of the face (not touching)

Override individual thresholds while keeping the preset defaults for the rest:

```python
config = FaceTouchConfig(
    sensitivity=FaceTouchSensitivity.MEDIUM,
    touch_distance_threshold=0.05,  # tighter distance
    confirmation_frames=4,          # more frames needed
)
```

## False-Positive Filtering

The filter applies multiple layers to reduce false alerts:

| Layer | What it catches |
|-------|----------------|
| Face bounding box | Hands beside the face, not on it |
| Distance threshold | Hands near but not close enough |
| Z-depth filter | Hands held in front of face (hovering) |
| Confirmation window | Momentary noise, hand passing by |
| Cooldown | Rapid re-triggers for the same zone |

## Performance

- **MediaPipe** runs locally — no cloud APIs, no network latency
- **`every_n_frames`** controls CPU usage: at default `3` with 15fps video, detection runs ~5 times/second
- **First frame** is slower (model loading ~1-2s), subsequent frames take ~15-30ms on modern CPUs
- Increase `every_n_frames` to reduce CPU load at the cost of reaction time

```python
config = FaceTouchConfig(every_n_frames=5)  # ~3 analyses/sec at 15fps
```

## The `ON_VIDEO_DETECTION` Hook

`FaceTouchFilter` emits `VideoDetectionEvent` with `kind="face_touch"`. This is a generic hook trigger shared by all video detection filters (YOLO, face touch, future detections).

The event payload:

```python
@dataclass
class VideoDetectionEvent:
    kind: str                      # "face_touch"
    session: VideoSession | None   # populated by channel
    labels: list[str]              # ["left_cheek"] or ["chin", "mouth"]
    confidence: float              # 0.0–1.0
    metadata: dict[str, Any]       # {"zone": "left_cheek", "hand": "right", ...}
    timestamp: datetime
    frame_sequence: int
```

The `metadata` dict for face touch events contains:

| Key | Type | Description |
|-----|------|-------------|
| `zone` | `str` | Face zone name (`"left_cheek"`, `"chin"`, etc.) |
| `hand` | `str` | `"left"` or `"right"` |
| `distance` | `float` | Normalized fingertip-to-zone distance |
| `touch_count` | `int` | Session cumulative touch count |

## Combining with Voice Alerts

Pair with a `VoiceChannel` to play audio alerts — the RoomKit equivalent of FaceTouchGuard's `afplay` clips:

```python
from roomkit import VoiceChannel

voice = VoiceChannel("voice", backend=voice_backend, tts=tts_provider)
kit.register_channel(voice)

@kit.hook(HookTrigger.ON_VIDEO_DETECTION, execution=HookExecution.ASYNC)
async def alert(event: VideoDetectionEvent, ctx):
    if event.kind == "face_touch":
        voice_ch = kit.get_channel("voice")
        await voice_ch.say("Stop touching your face!")
```

## Testing with MockFaceTouchFilter

Use `MockFaceTouchFilter` to test hook handlers without MediaPipe:

```python
from roomkit.video.pipeline.filter.mock_face_touch import MockFaceTouchFilter
from roomkit.video.events import VideoDetectionEvent

mock = MockFaceTouchFilter(events_at={
    5: [VideoDetectionEvent(
        kind="face_touch",
        labels=["left_cheek"],
        confidence=0.85,
        metadata={"zone": "left_cheek", "hand": "right", "touch_count": 1},
    )],
})

pipeline = VideoPipelineConfig(filters=[mock])
```

## API Reference

| Class | Module | Description |
|-------|--------|-------------|
| `FaceTouchFilter` | `roomkit.video.pipeline.filter.mediapipe_face_touch` | MediaPipe-based detection filter |
| `FaceTouchConfig` | `roomkit.video.pipeline.filter.mediapipe_face_touch` | Configuration with sensitivity presets |
| `FaceTouchSensitivity` | `roomkit.video.pipeline.filter.mediapipe_face_touch` | `LOW`, `MEDIUM`, `HIGH` presets |
| `FaceZone` | `roomkit.video.pipeline.filter.mediapipe_face_touch` | Face zone enum |
| `MockFaceTouchFilter` | `roomkit.video.pipeline.filter.mock_face_touch` | Mock for testing |
| `VideoDetectionEvent` | `roomkit.video.events` | Detection event payload |
| `FilterEvent` | `roomkit.video.pipeline.filter.base` | Generic filter event wrapper |

## See Also

- [Video & Vision guide](video-vision.md) — vision AI with VLMs (Gemini, OpenAI)
- [Video Overview](video-overview.md) — video pipeline architecture
- [`examples/face_touch_guard.py`](https://github.com/anthropics/roomkit/blob/main/examples/face_touch_guard.py) — runnable example
