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

A participant who never speaks produces no file: the recording opens on the first frame, which is also where that track's own format is known.

### Each track is recorded in the format it was published in

Participants negotiate their audio format with the SFU separately, and nothing obliges them to agree — a phone dial-in at 8 kHz 8-bit and a studio microphone at 48 kHz stereo are an ordinary pair in one meeting. So the format belongs to the track, not to the conference: the recording is opened on the first frame and `RecordingTrack` carries what that frame actually was.

```python
@kit.hook(HookTrigger.ON_RECORDING_STARTED, execution=HookExecution.ASYNC)
async def recording_started(event: ConferenceRecordingStarted, ctx) -> None:
    print(f"{event.participant_id}: {event.sample_rate} Hz, {event.channels} ch, {event.codec}")
```

`codec` names the sample width the way FFmpeg does — `pcm_s8`, `pcm_s16le`, `pcm_s32le`, signed little-endian throughout. Four-byte samples are `pcm_s32le` and never float: `AudioFrame` carries a width, not a format, and the framework's resamplers and pipeline stages are integer throughout. A backend holding float samples converts them before handing them over.

**A recording carries one format.** A container fixes its streams at the first write, so a frame arriving in a format other than the one its recording was opened on has nowhere honest to go. It is not written: it is logged once for the track and counted in `recording_dropped_frames` below. Writing it would be worse than losing it — PCM carries no description of itself, so the frame would be decoded as the header claims, and the file would open, play wrong, and report nothing.

If you write your own `MediaRecorder`, this is the contract from your side: `on_data` hands you bytes and nothing else, so `track.codec`, `track.sample_rate` and `track.channels` are the only things that say how to read them. `channels` is `None` where a caller did not state it, which means mono.

Recording counts as collection, so it stops when the room binding stops permitting it (`kit.set_access(room, "conf", Access.NONE)`), and `channel.info()["rooms"][room]["recording_active"]` answers whether a given conference is being recorded right now.

`ConferenceRecordingConfig(mode="egress")` — delegating to the SFU, the only way to obtain a *composed* grid or active-speaker video — is specified but not implemented, and is refused rather than silently ignored.

### Nothing the recorder does happens on the frame callback

A conference backend hands each frame to its subscribers in sequence, so anything slow done where a frame arrives is time every *other* participant's audio waits for. Encoding and writing a file is exactly that — and so is *creating* one, and so is closing it. A conference recording therefore does none of them there: the frame is queued and the callback returns, and the recording's whole life — the open, the writes, the close — happens on a task of its own.

Off the loop's thread, too. Every call in `MediaRecorder` is synchronous — each blocks for as long as the storage takes — so a queue alone would only move the block a few microseconds later. **Your recorder is called from a worker thread, on every method**, and it must not assume otherwise (RFC §12.11). What the framework promises in return is serialization per recording: the calls belonging to one handle are ordered and never overlap, so per-recording state needs no locking of its own. Different handles may be used at the same time, which is what keeps one participant's slow disk one participant's problem.

Two consequences worth knowing:

- **A recording opens a moment after the frame that decided to record it**, and its writes land a moment after that. A test that asserts on what a recorder received has to wait for them (`await channel._recorder.drain()`); in production the flush happens before a recording is finalized, so a closed file is never missing frames that were still in flight. `ON_RECORDING_STARTED` fires when the recorder has actually accepted the recording, which is also the first moment it has an id to report.
- **A recording that falls behind drops frames rather than memory.** Each track's write backlog is bounded by `max_queued_frames` (the same bound the transcription lanes use — default 100 frames, about two seconds), and a full backlog drops its **oldest** frame. That leaves a gap in the file rather than an ever-growing queue and an ever-growing lag.

```python
channel.info()["rooms"][room]["recording_dropped_frames"]  # 0 when nothing was lost
```

Frames a failed write lost — a full disk, a codec that refuses one frame — are counted there too, and the log says which of the two it was. None of it reaches the transcription path: a conference that cannot be recorded goes on being transcribed, and a track whose recording could not be opened at all is logged once rather than retried on every frame.

### Finding the files

A recording nobody can locate is not much of a recording, so the channel reports each one: `ON_RECORDING_STARTED` when a track's recording opens, `ON_RECORDING_STOPPED` when it closes with the result.

```python
from roomkit import (
    ConferenceRecordingStarted,
    ConferenceRecordingStopped,
    HookExecution,
    HookTrigger,
)

@kit.hook(HookTrigger.ON_RECORDING_STARTED, execution=HookExecution.ASYNC)
async def recording_started(event: ConferenceRecordingStarted, ctx) -> None:
    print(f"{event.participant_id} is being recorded ({event.kind}, {event.sample_rate} Hz)")

@kit.hook(HookTrigger.ON_RECORDING_STOPPED, execution=HookExecution.ASYNC)
async def recording_stopped(event: ConferenceRecordingStopped, ctx) -> None:
    await archive(
        room_id=event.room_id,
        participant_id=event.participant_id,
        track_id=event.track_id,
        location=event.url,           # where the recorder wrote it
        duration=event.duration_seconds,
        size=event.size_bytes,
    )
```

The same pair is emitted as framework events — `recording_started` and `recording_stopped`, carrying `channel_id`, `track_id`, `participant_id` and `url` — for observers that watch the bus rather than register hooks:

```python
@kit.on("recording_stopped")
async def on_stopped(event) -> None:
    print(event.data["url"])
```

**One report per track, not one per conference.** The tracks of a meeting do not end together: a participant who leaves halfway through has a finished file while the meeting runs on, and holding those results back for a single report at the end would deliver them after the point you could act on them. An integrator that wants the meeting's full list accumulates it by room — which is state it was going to keep anyway.

A track that stayed silent reports nothing at all, since it produced no file. Both hooks are ASYNC: a recording already written is a fact, not a decision. Egress mode fires neither — it carries no result contract (RFC §12.10.8).

A runnable end-to-end walkthrough, against the mock backend and recorder, is in [`examples/conference_recording_result.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/conference_recording_result.py).

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
