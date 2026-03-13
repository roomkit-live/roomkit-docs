# Room-Level Media Recording

Muxes audio and video from multiple channels in a room into a single MP4 file. Unlike channel-level recorders (WAV, PyAV video) that capture a single stream, room-level recording combines all media tracks into one output — the production path for recording conversations with both voice and video.

## Installation

```bash
pip install roomkit[video]          # av + numpy (PyAV muxer)
pip install roomkit[local-audio]    # sounddevice (mic capture)
pip install roomkit[local-video]    # opencv (webcam capture)
```

## Quick start

```python
from roomkit import (
    ChannelRecordingConfig,
    MediaRecordingConfig,
    RoomKit,
    VideoChannel,
    VoiceChannel,
)
from roomkit.recorder import RoomRecorderBinding
from roomkit.recorder.pyav import PyAVMediaRecorder

# 1. Create recorder + config
recorder = PyAVMediaRecorder()
config = MediaRecordingConfig(storage="./recordings", video_codec="auto")

# 2. Mark which channels contribute audio/video
voice = VoiceChannel("voice", backend=audio_backend, pipeline=pipeline,
                     recording=ChannelRecordingConfig(audio=True))
video = VideoChannel("video", backend=video_backend,
                     recording=ChannelRecordingConfig(video=True))

# 3. Create room with recorder binding
room = await kit.create_room(
    room_id="my-room",
    recorders=[RoomRecorderBinding(recorder=recorder, config=config, name="main")],
)

# 4. Connect participants — recording starts automatically
voice_session = await kit.connect_voice(room.id, "user-1", "voice")
video_session = await kit.connect_video(room.id, "user-1", "video")
```

Recording starts when all registered tracks (audio + video) have received their first frame. It stops when the room is closed or `close_room()` is called.

## Recording layers

RoomKit has three independent recording layers:

| Layer | Recorder | Purpose | Output |
|-------|----------|---------|--------|
| Audio pipeline | `WavFileRecorder` | Debug raw audio | `.wav` per session |
| Video pipeline | `PyAVVideoRecorder` | Debug raw video | `.mp4` per session |
| **Room** | **`MediaRecorder`** | **Production A/V** | **Single `.mp4` per room** |

All three can run simultaneously without interference.

## Configuration

### MediaRecordingConfig

Controls the output file format and encoding:

```python
from roomkit import MediaRecordingConfig

config = MediaRecordingConfig(
    storage="./recordings",    # Output directory (created automatically)
    video_codec="auto",        # auto, libx264, h264_nvenc, libx265
    video_fps=30,              # Stream frame rate (PTS resolution)
    audio_codec="aac",         # Audio codec
    audio_sample_rate=16000,   # Audio sample rate (Hz)
    format="mp4",              # Container format
)
```

| Field | Default | Description |
|-------|---------|-------------|
| `storage` | `./recordings` | Output directory path |
| `video_codec` | `auto` | Tries NVENC first, falls back to libx264 |
| `video_fps` | `30` | Video stream rate for PTS resolution |
| `audio_codec` | `aac` | Audio codec (AAC recommended for MP4) |
| `audio_sample_rate` | `16000` | Audio sample rate in Hz |
| `format` | `mp4` | Container format |

### ChannelRecordingConfig

Controls which media types from a channel are fed to room recorders:

```python
from roomkit import ChannelRecordingConfig

# Audio only
voice = VoiceChannel("voice", ..., recording=ChannelRecordingConfig(audio=True))

# Video only
video = VideoChannel("video", ..., recording=ChannelRecordingConfig(video=True))

# Both (future: screen share)
video = VideoChannel("video", ...,
    recording=ChannelRecordingConfig(video=True, screen_share=True))
```

## A/V sync

Audio and video PTS are both derived from `time.monotonic()` at frame acquisition time, referenced to a shared origin set after all codec streams are initialized. This ensures:

- Audio and video stay aligned regardless of pipeline latency differences
- Playback speed matches real time regardless of configured FPS vs actual capture rate
- NVENC initialization delay (which can block 200-500ms) doesn't cause offset between tracks

## Custom recorder

Implement the `MediaRecorder` ABC to write to a custom backend (cloud storage, streaming server, etc.):

```python
from roomkit.recorder.base import (
    MediaRecorder,
    MediaRecordingConfig,
    MediaRecordingHandle,
    MediaRecordingResult,
    RecordingTrack,
)

class MyCloudRecorder(MediaRecorder):
    @property
    def name(self) -> str:
        return "cloud"

    def on_recording_start(self, config: MediaRecordingConfig) -> MediaRecordingHandle:
        # Initialize upload session
        ...

    def on_recording_stop(self, handle: MediaRecordingHandle) -> MediaRecordingResult:
        # Finalize and return URL
        ...

    def on_track_added(self, handle: MediaRecordingHandle, track: RecordingTrack) -> None:
        ...

    def on_track_removed(self, handle: MediaRecordingHandle, track: RecordingTrack) -> None:
        ...

    def on_data(self, handle, track, data: bytes, timestamp_ms: float | None) -> None:
        # Stream audio/video chunks to cloud
        ...
```

## File naming

Output files are named `room_{handle_id}_{timestamp}.mp4` where `handle_id` is a random 12-character hex string and `timestamp` is `YYYYMMDDTHHMMSS` in UTC.

## Testing

Use `MockMediaRecorder` for tests — it stores tracks and data chunks in memory:

```python
from roomkit.recorder import MockMediaRecorder

recorder = MockMediaRecorder()
# ... run test ...
assert len(recorder.tracks) == 2      # audio + video
assert len(recorder.chunks) > 0       # data was received
assert recorder.results[0].size_bytes > 0
```

## Example

See [`examples/room_media_recorder.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/room_media_recorder.py) for a complete runnable example with mic + webcam recording.

```bash
uv run python examples/room_media_recorder.py
uv run python examples/room_media_recorder.py --duration 10 --fps 30
uv run python examples/room_media_recorder.py --output ./my_recordings --device 0
```
