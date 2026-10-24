# Video Backends (RTP & SIP)

Transport backends for sending and receiving real-time video over RTP, with optional SIP signaling. These extend the voice backends with parallel video RTP sessions, so a single backend handles both audio and video for a call.

## Architecture

Both are `AudioVideoBackend`s: voice backends whose calls may carry video (RFC §12.8.3).

```
RTPVideoBackend(RTPVoiceBackend, AudioVideoBackend)    backend.video → its calls' video
SIPVideoBackend(SIPVoiceBackend, AudioVideoBackend)    backend.video → its calls' video
```

The call is the voice session: the backend's own `on_session_ready` and `on_client_disconnected` report every call, audio-only ones included. The video of a call is served by `backend.video`, a `VideoBackend` whose sessions carry the call's id, so you can look up either from the same ID. A single `connect()` or incoming INVITE creates both audio and video RTP sessions.

The backend is not itself a `VideoBackend`: the two interfaces share `on_session_ready` and `on_client_disconnected` with different meanings. Hand `backend.video` to whatever takes a `VideoBackend`, e.g. `VideoChannel("cam", backend=backend.video)`.

```
Inbound:
  RTP video packets → aiortp VideoRTPSession → H.264 depacketization
    → VideoFrame(codec="h264", data=nal_bytes, keyframe, timestamp_ms)
    → backend.video.on_video_received callback

Outbound:
  backend.video.send_video(session, bytes | AsyncIterator[VideoChunk])
    → H.264 packetization → RTP video packets → remote
```

## RTPVideoBackend

Direct RTP video transport with pre-configured endpoints. No SIP signaling — you supply the addresses.

### Quick start

```python
from roomkit.video.backends import RTPVideoBackend
from roomkit import VoiceChannel
from roomkit.voice.pipeline import AudioPipelineConfig

backend = RTPVideoBackend(
    local_addr=("0.0.0.0", 10000),
    remote_addr=("192.168.1.100", 20000),
    video_local_addr=("0.0.0.0", 10002),
    video_remote_addr=("192.168.1.100", 20002),
)

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend, pipeline=pipeline)

# Pull model: kit.join() creates the session, binds it, and wires recording
session = await kit.join("room-1", "voice", participant_id="user-1")
# Audio + video RTP sessions are now active

video_session = backend.video.get_session(session.id)
```

Install with:

```
pip install roomkit[rtp]
```

### Constructor parameters

All [RTPVoiceBackend parameters](rtp-backend.md) plus:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `video_local_addr` | `(str, int)` | `("0.0.0.0", 0)` | Host and port to bind video RTP. |
| `video_remote_addr` | `(str, int) \| None` | `None` | Host and port to send video RTP to. Can be overridden per-session. |
| `video_payload_type` | `int` | `96` | RTP payload type for video (96 = H.264). |
| `video_clock_rate` | `int` | `90000` | Video RTP clock rate in Hz. |

### Per-session video remote address

Like the voice backend, you can omit `video_remote_addr` from the constructor and supply it per-session:

```python
backend = RTPVideoBackend(
    local_addr=("0.0.0.0", 10000),
    remote_addr=("10.0.0.1", 20000),
    video_local_addr=("0.0.0.0", 10002),
)

session = await kit.join(
    "room-1", "voice",
    participant_id="user-1",
    metadata={"video_remote_addr": ("10.0.0.1", 20002)},
)
```

If neither the constructor nor metadata provides a video remote address, `connect()` raises `ValueError`.

## SIPVideoBackend

SIP-based video transport with full call signaling. Extends `SIPVoiceBackend` — when an incoming INVITE contains `m=video`, the backend negotiates both audio and video codecs and creates parallel RTP sessions. Audio-only calls work transparently.

### Quick start

```python
from roomkit.video.backends import SIPVideoBackend
from roomkit import VoiceChannel
from roomkit.voice import VoiceSession

backend = SIPVideoBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
    supported_video_codecs=["H264", "VP8"],
)

# Video frame callback
def on_video(session, frame):
    print(f"Video: codec={frame.codec}, keyframe={frame.keyframe}, seq={frame.sequence}")

backend.video.on_video_received(on_video)

# Route incoming calls
async def on_call(session: VoiceSession):
    has_video = session.metadata.get("has_video", False)
    print(f"Call from {session.metadata['caller']}, video={has_video}")

backend.on_call(on_call)
await backend.start()
```

Install with:

```
pip install roomkit[sip]
```

### Constructor parameters

All [SIPVoiceBackend parameters](sip-backend.md) plus:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `supported_video_codecs` | `list[str] \| None` | `["H264"]` | Video codec names to accept in SDP negotiation. |

### How A/V calls work

```
PBX/SIP Trunk                    SIPVideoBackend
─────────────                    ───────────────
  INVITE ──────────────────────► receives call
  (SDP with m=audio + m=video)     │
                                   ├── Audio SDP negotiation (CallSession)
                                   ├── Video SDP negotiation (VideoCallSession)
                                   ├── Combined SDP answer (audio + video)
                                   │
  ◄──── 100 Trying                 │
  ◄──── 180 Ringing                │
  ◄──── 200 OK (combined SDP)     ├── on_call fires (has_video=True)
                                   │
  RTP audio ◄──────────────────►  audio pipeline
  RTP video ◄──────────────────►  video.on_video_received
                                   │
  BYE ─────────────────────────► cleanup (audio + video)
```

When the INVITE has no `m=video`, the call is handled as audio-only by the parent `SIPVoiceBackend` — no video session is created.

### Graceful fallback

If video SDP negotiation fails (no matching codecs), the call proceeds audio-only. The `has_video` metadata flag will be `False`:

```python
async def on_call(session: VoiceSession):
    if session.metadata.get("has_video"):
        video_session = backend.video.get_session(session.id)
        # wire up video processing
    else:
        # audio-only call
        pass
```

## Receiving video

Register a callback to receive inbound video frames:

```python
from roomkit.video import VideoFrame

def on_video(session, frame: VideoFrame):
    print(f"codec={frame.codec}, size={len(frame.data)}, "
          f"keyframe={frame.keyframe}, seq={frame.sequence}, "
          f"ts={frame.timestamp_ms:.1f}ms")

backend.video.on_video_received(on_video)
```

`VideoFrame` fields:

| Field | Type | Description |
|---|---|---|
| `data` | `bytes` | Raw NAL unit data (H.264) or encoded frame. |
| `codec` | `str` | Codec identifier (e.g. `"h264"`). |
| `timestamp_ms` | `float \| None` | Presentation timestamp in milliseconds. |
| `keyframe` | `bool` | Whether this is an IDR/keyframe. |
| `sequence` | `int` | Monotonically increasing frame counter per session. |

## Sending video

Send video frames to the remote party:

```python
# Send raw bytes (single NAL unit)
await backend.video.send_video(video_session, nal_bytes)

# Send a stream of VideoChunks
from roomkit.video import VideoChunk

async def video_source():
    yield VideoChunk(data=nal_bytes, keyframe=True, timestamp_ms=0)
    yield VideoChunk(data=nal_bytes, keyframe=False, timestamp_ms=33)

await backend.video.send_video(video_session, video_source())
```

## Session queries

```python
# Get video session by ID (same ID as voice session)
video_session = backend.video.get_session(session.id)

# List all video sessions in a room
sessions = backend.video.list_sessions("room-1")
```

## Callbacks

| Callback | Description |
|---|---|
| `backend.on_session_ready(cb)` | A call's audio path is live (every call), with its `VoiceSession`. |
| `backend.on_client_disconnected(cb)` | A call ended (every call), with its `VoiceSession`. |
| `backend.video.on_session_ready(cb)` | A call's video path is live (calls with video only), with its `VideoSession`. |
| `backend.video.on_client_disconnected(cb)` | A call's video ended, which is when its call ends. |
| `backend.video.on_video_received(cb)` | Inbound video frames (NAL units). |

`AudioVideoChannel` subscribes to both: it fires `ON_SESSION_STARTED` for every call and `ON_VIDEO_SESSION_STARTED` / `ON_VIDEO_SESSION_ENDED` once per call with video.

The video methods that used to sit on the backend itself (`get_video_session()`, `list_video_sessions()`, `on_video_received()`, `add_video_tap()`, `send_video()`, `send_video_sync()`, `request_keyframe()`, `set_video_passthrough()`, `video_capabilities`) still work and emit a `DeprecationWarning`; use `backend.video` instead.

## RTP vs SIP video backend

| Feature | RTP video | SIP video |
|---|---|---|
| SIP signaling | Not included | Built-in (INVITE, BYE, SDP) |
| SDP negotiation | Manual codec config | Automatic codec selection |
| Session creation | Manual via `connect()` | Automatic on INVITE |
| Remote address | Must be configured | From SDP offer |
| Audio-only fallback | N/A (always creates video) | Automatic if no `m=video` |
| Dependencies | `aiortp` | `aiosipua[rtp]` |
| Use case | Direct RTP endpoints | PBX/trunk integration |

## Example

See [`examples/sip_video_call.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/sip_video_call.py) for a complete SIP A/V call handler with incoming call routing and video frame logging.
