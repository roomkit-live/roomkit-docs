# PyAV Video Recorder

A production video recorder that writes H.264/H.265 compressed MP4 files using FFmpeg via [PyAV](https://pyav.org/). Produces 10-50x smaller files than the OpenCV recorder and supports NVIDIA hardware encoding (NVENC).

## Installation

```bash
pip install roomkit[video]          # av + numpy
pip install roomkit[local-video]    # opencv for webcam capture
```

## Quick start

```python
from roomkit import VideoChannel, VideoPipelineConfig
from roomkit.video.backends.local import LocalVideoBackend
from roomkit.video.recorder import VideoRecordingConfig
from roomkit.video.recorder.pyav import PyAVVideoRecorder

backend = LocalVideoBackend(device=0, fps=15)
recorder = PyAVVideoRecorder()
config = VideoRecordingConfig(storage="./recordings", codec="auto")

video = VideoChannel(
    "video",
    backend=backend,
    pipeline=VideoPipelineConfig(recorder=recorder, recording_config=config),
)
```

Recording starts automatically when a video session binds and stops when it unbinds. Every received frame is encoded to H.264 and muxed into the MP4 container.

## Codec selection

The `codec` field in `VideoRecordingConfig` controls encoding:

| Value | Encoder | Description |
|-------|---------|-------------|
| `auto` (default) | NVENC or libx264 | Tries `h264_nvenc` first, falls back to `libx264` |
| `libx264` | x264 (CPU) | Software H.264 — works everywhere, good compression |
| `h264_nvenc` | NVIDIA NVENC | Hardware H.264 — very fast, requires NVIDIA GPU |
| `libx265` | x265 (CPU) | Software H.265/HEVC — best compression, slower |

```python
# Explicit software encoding
config = VideoRecordingConfig(codec="libx264")

# Force NVIDIA hardware encoding
config = VideoRecordingConfig(codec="h264_nvenc")

# Auto-detect (recommended)
config = VideoRecordingConfig(codec="auto")
```

When `codec="auto"`, the recorder probes for NVENC at startup. If an NVIDIA GPU with encoding support is available, it uses hardware encoding; otherwise it falls back to `libx264`. This happens once per recording start, not per frame.

## Recorder comparison

| Recorder | Codec | Compression | File size (10s @ 15fps) | Install |
|----------|-------|-------------|-------------------------|---------|
| `OpenCVVideoRecorder` | mp4v (MPEG-4 Part 2) | Low | ~5-10 MB | `roomkit[local-video]` |
| `PyAVVideoRecorder` | H.264 (libx264) | High | ~100-500 KB | `roomkit[video]` |
| `PyAVVideoRecorder` | H.265 (libx265) | Very high | ~50-300 KB | `roomkit[video]` |
| `MockVideoRecorder` | None | N/A | In-memory | Built-in |

## Configuration

All options are set via `VideoRecordingConfig`:

```python
from roomkit.video.recorder import VideoRecordingConfig

config = VideoRecordingConfig(
    storage="./recordings",   # Output directory (created automatically)
    format="mp4",             # Container format: mp4, avi, mkv
    codec="auto",             # See codec selection above
    fps=15.0,                 # Output frame rate
)
```

## Frame format

The PyAV recorder accepts raw (uncompressed) frames only:

- `raw_rgb24` — standard RGB pixel data
- `raw_bgr24` — BGR pixel data (OpenCV native order)

Encoded frames (h264, vp8, etc.) are skipped with a debug log. The `LocalVideoBackend` produces `raw_bgr24` frames by default, which the recorder handles directly.

Frame dimensions are set lazily from the first received frame — no need to configure width/height in advance.

## File naming

Output files are named `{session_id}_{timestamp}.mp4` where the timestamp is `YYYYMMDDTHHMMSS` in UTC. Special characters in the session ID are replaced with underscores.

## Error handling

- **Missing `av` package**: `PyAVVideoRecorder()` raises `ImportError` with install instructions.
- **NVENC unavailable**: `codec="auto"` silently falls back to `libx264`. Explicit `codec="h264_nvenc"` will fail at `start()` if no NVIDIA GPU is available.
- **Zero-frame recording**: `stop()` on a recording with no frames closes the container cleanly without attempting to flush the encoder.

## Example

See [`examples/pyav_video_recorder.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/pyav_video_recorder.py) for a complete runnable example with codec selection and duration control.

```bash
uv run python examples/pyav_video_recorder.py
uv run python examples/pyav_video_recorder.py --codec h264_nvenc --duration 30
uv run python examples/pyav_video_recorder.py --output ./my_recordings --fps 30
```
