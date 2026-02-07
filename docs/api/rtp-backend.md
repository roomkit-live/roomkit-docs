# RTP Voice Backend

The RTP backend sends and receives voice audio over RTP for integration with PBX/SIP gateways or any system that speaks RTP.

Install with: `pip install roomkit[rtp]`

## Quick start

```python
from roomkit.voice.backends.rtp import RTPVoiceBackend

backend = RTPVoiceBackend(
    local_addr=("0.0.0.0", 10000),
    remote_addr=("192.168.1.100", 20000),
)
```

See the [full RTP example](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_rtp.py) for a complete runnable script.

## API Reference

::: roomkit.voice.backends.rtp.RTPVoiceBackend

## Capabilities

The RTP backend declares `DTMF_SIGNALING` (RFC 4733 out-of-band DTMF) and `INTERRUPTION` (cancel outbound audio mid-stream).

## DTMF

Inbound DTMF digits are received via RFC 4733 and delivered as `DTMFEvent` objects. Hook into them with `HookTrigger.ON_DTMF`:

```python
@kit.hook(HookTrigger.ON_DTMF, execution=HookExecution.ASYNC)
async def on_dtmf(event, ctx):
    print(f"DTMF: {event.digit}")
```

## Per-session remote address

Pass `metadata={"remote_addr": (host, port)}` to `connect()` to override the remote address per session, useful for multi-tenant PBX setups.
