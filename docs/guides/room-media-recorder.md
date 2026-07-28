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
from roomkit import RoomKit, VideoChannel, VoiceChannel
from roomkit.recorder import MediaRecordingConfig
from roomkit.recorder import RoomRecorderBinding
from roomkit.recorder.pyav import PyAVMediaRecorder

# 1. Create recorder + config
recorder = PyAVMediaRecorder()
config = MediaRecordingConfig(storage="./recordings", video_codec="auto")

# 2. Create channels — recording is automatic when the room has recorders
voice = VoiceChannel("voice", backend=audio_backend, pipeline=pipeline)
video = VideoChannel("video", backend=video_backend)

# 3. Create room with recorder binding
room = await kit.create_room(
    room_id="my-room",
    recorders=[RoomRecorderBinding(recorder=recorder, config=config, name="main")],
)

# 4. Join participants — recording starts automatically
# Previously connect_voice() / connect_video(), now unified as join()
voice_session = await kit.join(room.id, "voice", participant_id="user-1")
video_session = await kit.join(room.id, "video", participant_id="user-1")
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

A conference uses the `MediaRecorder` interface at a different granularity — one recording per track rather than one per room. See [Conference recording](#conference-recording).

## Configuration

### MediaRecordingConfig

Controls the output file format and encoding:

```python
from roomkit.recorder import MediaRecordingConfig

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

When a room has recorders, **all channels record automatically** — no per-channel configuration is needed. `ChannelRecordingConfig` is only required to **opt out** of recording specific media types on a channel:

```python
from roomkit.recorder import ChannelRecordingConfig

# Exclude video from this channel (audio still recorded)
voice = VoiceChannel("voice", ..., recording=ChannelRecordingConfig(video=False))

# Exclude screen share from recording
video = VideoChannel("video", ..., recording=ChannelRecordingConfig(screen_share=False))
```

## Conference recording

A conference records through the same `MediaRecorder` interface, but at a different granularity: **one recording per track**, not one per room. The recorder is passed to the channel rather than to `create_room()`.

```python
from roomkit import ConferenceRecordingConfig
from roomkit.channels.conference import ConferenceChannel
from roomkit.recorder.pyav import PyAVMediaRecorder

channel = ConferenceChannel(
    "conf",
    backend=backend,
    stt=stt,
    recorder=PyAVMediaRecorder(),
    recording=ConferenceRecordingConfig(storage="./recordings"),
)
```

Each subscribed track opens its own recording on its first frame, carrying that track alone with `RecordingTrack.participant_id` set to the publisher — so the output is per-participant and attributed, which is what a compliance recording of a meeting has to be. The bot's own published audio is recorded as one more attributed track (`bot:<session_id>`), never mixed into a participant's.

Per track rather than one file per room, because a conference gains participants while it runs. A room recording adds every track to a single container, and a container fixes its streams at the first write — a participant who joins ten minutes in cannot be added to it. Giving each track its own recording means no track is ever a late one.

A participant who never speaks produces no file: the recording opens on the first frame, which is also where that track's own sample rate is known.

Recording counts as collection, so it stops when the room binding stops permitting it (`kit.set_access(room, "conf", Access.NONE)`), and `channel.info()["rooms"][room]["recording_active"]` answers whether a given conference is being recorded right now.

`ConferenceRecordingConfig(mode="egress")` — delegating to the SFU, the only way to obtain a *composed* grid or active-speaker video — is specified but not implemented, and is refused rather than silently ignored.

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
