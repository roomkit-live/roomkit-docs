# SIP Voice Backend

A voice backend that handles the full SIP call lifecycle: listening for incoming INVITE requests, negotiating codecs via SDP, creating RTP sessions for audio, and managing call teardown (BYE/CANCEL). Uses `aiosipua` for SIP signaling and `aiortp` for media transport.

## Quick start

```python
from roomkit.voice.backends.sip import SIPVoiceBackend
from roomkit.voice import VoiceSession

backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
)

# Route incoming calls to rooms
def on_call(session: VoiceSession):
    room_id = session.metadata.get("room_id", session.id)
    print(f"Incoming call for room {room_id}")

backend.on_call(on_call)
backend.on_call_disconnected(lambda s: print(f"Call ended: {s.id}"))

await backend.start()
```

Install with:

```
pip install roomkit[sip]
```

This pulls in both `aiosipua` and `aiortp` transitively.

## How it works

Unlike the [RTP backend](rtp-backend.md) which requires manual address configuration and no SIP signaling, the SIP backend manages the complete call flow:

```
PBX/SIP Trunk                    SIPVoiceBackend
─────────────                    ───────────────
  INVITE ──────────────────────► receives call
  (SDP offer,                      │
   X-Room-ID,                      ├── SDP negotiation (codec selection)
   X-Session-ID)                   ├── RTP session creation
                                   │
  ◄──── 100 Trying                 │
  ◄──── 180 Ringing                │
  ◄──── 200 OK (SDP answer)       ├── on_call callback fires
                                   │
  RTP audio ◄──────────────────►  audio pipeline (VAD → STT → AI → TTS)
  DTMF (RFC 4733) ◄──────────►   │
                                   │
  BYE ─────────────────────────► on_call_disconnected callback fires
  ◄──── 200 OK                    cleanup
```

1. PBX sends an INVITE with SDP offer and optional X-headers
2. Backend negotiates codecs, sends 100 Trying → 180 Ringing → 200 OK
3. RTP session is created automatically from the negotiated SDP
4. `on_call` callback fires with a `VoiceSession` for the app to route
5. Audio flows through the pipeline (same as RTP backend)
6. When the remote party sends BYE, `on_call_disconnected` fires

## Constructor parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `local_sip_addr` | `(str, int)` | `("0.0.0.0", 5060)` | Host and port to bind the SIP UDP listener. |
| `local_rtp_ip` | `str` | `"0.0.0.0"` | IP address for RTP media binding. Use your server's actual IP in production. |
| `rtp_port_start` | `int` | `10000` | First port in the RTP allocation range. |
| `rtp_port_end` | `int` | `20000` | Last port in the RTP allocation range. |
| `supported_codecs` | `list[int] \| None` | `[0, 8]` | Codec payload types to accept (PCMU, PCMA). |
| `dtmf_payload_type` | `int` | `101` | RTP payload type for RFC 4733 DTMF events. |

## X-header routing

The backend extracts routing metadata from SIP X-headers set by the PBX/proxy:

| X-Header | Maps to | Fallback |
|---|---|---|
| `X-Room-ID` | `session.room_id` / `session.metadata["room_id"]` | Call-ID |
| `X-Session-ID` | `session.id` / `session.participant_id` | Caller URI |

All X-headers are available in `session.metadata["x_headers"]` as a dict.

**Kamailio example** — adding X-headers before forwarding to roomkit:

```
# kamailio.cfg
route[FORWARD_TO_ROOMKIT] {
    append_hf("X-Room-ID: $var(room_id)\r\n");
    append_hf("X-Session-ID: $ci\r\n");
    append_hf("X-Tenant-ID: $var(tenant)\r\n");
    t_relay("udp:10.0.0.5:5060");
}
```

## Callbacks

The SIP backend provides two additional callbacks beyond the standard `VoiceBackend` interface:

### `on_call(callback)`

Fired after an incoming INVITE is accepted and the RTP session is active. This is where you route the session to a room:

```python
async def handle_call(session: VoiceSession):
    room_id = session.metadata.get("room_id", session.id)
    await kit.create_room(room_id=room_id)
    await kit.attach_channel(room_id, "voice")
    voice.bind_session(session, room_id, binding)

backend.on_call(handle_call)
```

### `on_call_disconnected(callback)`

Fired when the remote party sends BYE:

```python
async def handle_disconnect(session: VoiceSession):
    await kit.disconnect_voice(session)
    await kit.close_room(session.room_id)

backend.on_call_disconnected(handle_disconnect)
```

### Standard callbacks

| Callback | Description |
|---|---|
| `on_audio_received(cb)` | Raw inbound audio frames from RTP. |
| `on_barge_in(cb)` | Barge-in detection (user speaks during TTS). |
| `on_dtmf_received(cb)` | RFC 4733 DTMF digits with duration. |

## Connecting sessions to rooms

Unlike other backends where you call `backend.connect()` to create a session, SIP sessions are created automatically during INVITE handling. Use `connect()` to look up a pre-created session by its ID:

```python
# In your on_call handler:
session = await backend.connect(
    room_id, participant_id, channel_id,
    metadata={"session_id": session.id},
)
```

## Disconnecting

Call `backend.disconnect(session)` to hang up from the server side. This sends a SIP BYE to the remote party and closes the RTP session:

```python
await backend.disconnect(session)
# SIP BYE is sent, RTP session is closed, session state → ENDED
```

## DTMF

DTMF works the same as the RTP backend — digits arrive out-of-band via RFC 4733 and integrate with the hook system:

```python
@kit.hook(HookTrigger.ON_DTMF, execution=HookExecution.ASYNC)
async def on_dtmf(event, ctx):
    print(f"DTMF digit: {event.digit}, duration: {event.duration_ms}ms")
```

## Capabilities

| Capability | Description |
|---|---|
| `DTMF_SIGNALING` | DTMF digits received out-of-band via RFC 4733. |
| `INTERRUPTION` | Outbound audio playback can be cancelled mid-stream (barge-in). |

## Audio flow

### Inbound

```
Remote → RTP packets → aiortp decode → PCM-16 LE
  → AudioFrame(sample_rate=8000, channels=1, sample_width=2)
  → on_audio_received → AudioPipeline inbound chain
```

### Outbound

```
TTS → AudioChunk stream or bytes → PCM-16 LE
  → 20ms RTP frames (160 samples at 8kHz)
  → CallSession.send_audio_pcm → aiortp encode → RTP packets → remote
```

## RTP port allocation

The backend allocates RTP ports sequentially in pairs (RTP + RTCP) starting at `rtp_port_start`. When the range is exhausted, it wraps around to the start. Each call uses one port pair.

For production, ensure your firewall allows UDP traffic on the configured port range.

## SIP vs RTP backend

| Feature | SIP backend | RTP backend |
|---|---|---|
| SIP signaling | Built-in (INVITE, BYE, CANCEL) | Not included |
| SDP negotiation | Automatic codec selection | Manual codec configuration |
| Session creation | Automatic on INVITE | Manual via `connect()` |
| Remote address | From SDP offer | Must be configured |
| Dependencies | `aiosipua[rtp]` | `aiortp` |
| Use case | PBX/trunk integration | Direct RTP endpoints |

## API Reference

See the [SIP Backend API Reference](../api/sip-backend.md) for auto-generated class documentation.

## Example

See [`examples/voice_sip.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_sip.py) for a complete runnable example with incoming call handling and cleanup.
