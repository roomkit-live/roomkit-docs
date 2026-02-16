# SIP Voice Backend

The SIP backend handles the full SIP call lifecycle: incoming INVITE → SDP negotiation → RTP media → BYE teardown. For PBX and SIP trunk integration.

Install with: `pip install roomkit[sip]`

## Quick start

```python
from roomkit.voice.backends.sip import SIPVoiceBackend

backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
)
backend.on_call(handle_incoming_call)
await backend.start()
```

See the [full SIP example](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_sip.py) for a complete runnable script.

## API Reference

::: roomkit.voice.backends.sip.SIPVoiceBackend

## Capabilities

The SIP backend declares `DTMF_SIGNALING` (RFC 4733 out-of-band DTMF) and `INTERRUPTION` (cancel outbound audio mid-stream).

## X-header routing

Room and session IDs are extracted from `X-Room-ID` and `X-Session-ID` SIP headers. All X-headers are available in `session.metadata["x_headers"]`.

## Callbacks

In addition to the standard `VoiceBackend` callbacks, the SIP backend provides:

- `on_call(callback)` — fired when an incoming INVITE is accepted
- `on_call_disconnected(callback)` — fired when the remote party sends BYE
