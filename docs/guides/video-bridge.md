# Video Bridging

Video bridging enables direct session-to-session video forwarding for
human-to-human video calls.  Video frames flow between participants
without decoding or re-encoding, preserving native codec quality and
minimizing latency.

## Quick Start

Enable video bridging by passing `bridge=True` to `VideoChannel` or
`video_bridge=True` to `AudioVideoChannel`:

```python
from roomkit import RoomKit, AudioVideoChannel
from roomkit.video.bridge import VideoBridgeConfig
from roomkit.video.backends.sip import SIPVideoBackend
from roomkit.voice.pipeline import AudioPipelineConfig

backend = SIPVideoBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="0.0.0.0",
    supported_video_codecs=["H264", "VP8", "VP9"],
)

av = AudioVideoChannel(
    "av",
    backend=backend,
    pipeline=AudioPipelineConfig(),
    bridge=True,           # audio bridge (from VoiceChannel)
    video_bridge=True,     # video bridge
)

kit = RoomKit()
kit.register_channel(av)
```

When two participants join the same room and are bound to the channel,
video from each is forwarded to the other automatically.

## How It Works

```
Session A camera → Backend → Pipeline → VideoBridge.forward()
                                              │
                                              └─→ send_video_sync → Session B display

Session B camera → Backend → Pipeline → VideoBridge.forward()
                                              │
                                              └─→ send_video_sync → Session A display
```

1. **Inbound video** from each session passes through the video pipeline
   (decoder, resizer, transforms, filters) if configured.
2. **Recorder taps** and **media taps** receive the processed frame.
3. **VideoBridge** receives the frame and forwards it to all other
   sessions in the same room.
4. **Vision analysis** (if configured) runs in parallel — bridging and
   vision are independent paths.

## Configuration

### VideoBridgeConfig

```python
from roomkit.video.bridge import VideoBridgeConfig

config = VideoBridgeConfig(
    max_participants=10,           # Max sessions per room (default: 10)
    forwarding_strategy="forward", # Direct forwarding (only mode for now)
)

av = AudioVideoChannel("av", backend=backend, video_bridge=config)
```

| Parameter | Default | Description |
|---|---|---|
| `enabled` | `True` | Whether bridging is active. Set `False` to pause forwarding. |
| `max_participants` | `10` | Maximum bridged sessions per room. Raises `RuntimeError` if exceeded. |
| `forwarding_strategy` | `"forward"` | Direct frame forwarding. N-party compositing (`"composite"`) is planned for a future release. |
| `keyframe_interval_s` | `5.0` | Send a PLI to every session in the room this often, so a decoder stuck after packet loss recovers. `0` disables. |
| `keyframe_wait_s` | `1.0` | How long a receiver holds delta frames from a sender that has not produced a keyframe yet. `0` forwards unconditionally. |

### Waiting for a keyframe

A decoder handed delta frames from the middle of a stream renders garbage until
the next keyframe, so the bridge holds a sender's deltas until that sender has
delivered one keyframe *to that receiver*. Joining a room already requests a
keyframe from every peer, so a cooperating sender clears the gate almost
immediately and a late joiner starts on a clean picture instead of a smear.

The wait is bounded on purpose. A sender that never answers PLI — a B2BUA that
does not relay RTCP feedback, for instance — would otherwise leave the receiver
black forever, which is worse than a briefly corrupt picture. After
`keyframe_wait_s` the bridge logs a warning and stops gating that pair:

```
VideoBridge: no keyframe from 1a2b3c4d for 5e6f7a8b after 1.0s —
forwarding delta frames anyway (source may not answer PLI)
```

Seeing that warning routinely means the sender is not responding to keyframe
requests. Raising `keyframe_wait_s` only lengthens the black period; the fix is
on the signalling path. Set it to `0` when the topology is known never to relay
RTCP and you would rather not pay the delay at all.

### VideoChannel vs AudioVideoChannel

| Channel | Audio bridge | Video bridge | Use case |
|---------|-------------|-------------|----------|
| `VideoChannel` | — | `bridge=True` | Video-only forwarding |
| `AudioVideoChannel` | `bridge=True` | `video_bridge=True` | Full A/V call bridging (SIP, RTP) |

`AudioVideoChannel` extends `VoiceChannel`, so it supports audio
bridging natively.  Video bridging is an additional parameter.

## Video + Audio Bridge

For full A/V calls, enable both bridges:

```python
av = AudioVideoChannel(
    "av",
    backend=backend,
    pipeline=AudioPipelineConfig(),
    bridge=True,                    # audio: session-to-session forwarding
    video_bridge=VideoBridgeConfig(), # video: session-to-session forwarding
    stt=deepgram_provider,          # optional: transcription in parallel
)
```

Audio and video bridges operate independently.  Each has its own
session registry, filter/processor callbacks, and hook triggers.

## Frame Filtering

Use `set_bridge_filter()` to inspect or modify video frames before they
are forwarded.  The filter runs synchronously in the video callback
thread and must complete quickly:

```python
# Mute video from a specific participant
def mute_video(session, frame):
    if session.id == muted_session_id:
        return None  # drop frame
    return frame

video_channel.set_bridge_filter(mute_video)

# Remove the filter
video_channel.set_bridge_filter(None)
```

The filter receives `(source_session, VideoFrame)` and returns:

- The frame (unchanged or modified) to forward it
- `None` to drop the frame (hide the source for that frame)

## Frame Processor

Use `set_frame_processor()` on the `VideoBridge` directly for
per-target frame transformation.  The processor receives the target
session and the source frame, and returns a transformed `VideoFrame`:

```python
from roomkit.video.video_frame import VideoFrame

def downscale_for_mobile(target_session, frame):
    if target_session.metadata.get("client") == "mobile":
        # Return a smaller frame for mobile clients
        return resize(frame, width=320, height=240)
    return frame

channel._bridge.set_frame_processor(downscale_for_mobile)
```

## Hooks

### BEFORE_BRIDGE_VIDEO

The `BEFORE_BRIDGE_VIDEO` hook fires for each video frame before it is
forwarded to other sessions.  It supports `HookResult.block()` to drop
individual frames:

```python
@kit.hook(HookTrigger.BEFORE_BRIDGE_VIDEO)
async def monitor_bridge(event, ctx):
    # event.session — source session
    # event.frame — the VideoFrame about to be forwarded
    # event.room_id — the room where bridging is active

    if should_mute_video(event.session):
        return HookResult.block(reason="video_muted")
    return HookResult.allow()
```

!!! tip "Performance: fast path"
    When no `BEFORE_BRIDGE_VIDEO` hooks are registered, the bridge
    forwards frames directly in the video callback thread with zero
    overhead.  When hooks are registered, frames are routed through
    the event loop for hook evaluation.

For synchronous, low-latency frame filtering, use
`set_bridge_filter()` instead.

### Other Video Hooks

These hooks fire normally during bridge mode:

- `ON_VIDEO_SESSION_STARTED` / `ON_VIDEO_SESSION_ENDED` — session lifecycle
- `ON_VISION_RESULT` — if a VisionProvider is configured
- `ON_VIDEO_TRACK_ADDED` / `ON_VIDEO_TRACK_REMOVED` — track changes

## Session Lifecycle

Sessions are registered with the bridge when bound to the channel
and unregistered when unbound:

```python
# Join: binds session and adds to bridge
# Previously av.bind_session() / av.unbind_session()
await kit.join("room-1", "av", session=session)

# Leave: unbinds session and removes from bridge
await kit.leave(session)
```

## Thread Safety

All bridge operations are thread-safe.  `VideoBridge.forward()` is
called from video callback threads (the same context as
`on_video_received`).  Internal state is protected by a
`threading.Lock`.

## Comparison with Audio Bridging

| | AudioBridge | VideoBridge |
|---|---|---|
| **Strategies** | `"forward"` + `"mix"` (N-party) | `"forward"` only (compositing deferred) |
| **Per-target adaptation** | Sample rate resampling | None yet (transcoding planned) |
| **Hook** | `BEFORE_BRIDGE_AUDIO` | `BEFORE_BRIDGE_VIDEO` |
| **Channel param** | `bridge=True` | `bridge=True` (VideoChannel) / `video_bridge=True` (AudioVideoChannel) |
| **Sync send** | `send_audio_sync()` | `send_video_sync()` |

## Examples

### SIP Audio+Video Bridge

`examples/sip_video_bridge.py` — Two SIP callers bridged with audio
and video forwarding.  Both participants see and hear each other at
native quality.

```bash
uv run python examples/sip_video_bridge.py
```

Then send two SIP INVITEs with `m=audio` + `m=video` to port 5060.

### SIP Audio Bridge (Audio Only)

`examples/voice_sip_bridge.py` — Two SIP callers bridged with audio
only.  Deepgram STT transcribes both sides in real time.

```bash
DEEPGRAM_API_KEY=... uv run python examples/voice_sip_bridge.py
```
