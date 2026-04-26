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
| `supported_codecs` | `list[int] \| None` | `[9, 0, 8]` | Codec payload types to accept (G.722, PCMU, PCMA). |
| `dtmf_payload_type` | `int` | `101` | RTP payload type for RFC 4733 DTMF events. |
| `jitter_capacity` | `int` | `32` | Max packets the RTP jitter buffer can hold (~640 ms at 20 ms/packet). |
| `jitter_prefetch` | `int` | `0` | Packets to accumulate before starting playout. 0 = start immediately. |
| `skip_audio_gaps` | `bool` | `True` | Skip gaps in the RTP stream rather than filling with silence. |
| `rtp_inactivity_timeout` | `float` | `30.0` | Seconds of RTP silence before forcing session disconnect (0 to disable). |

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

## Authentication

Inbound INVITEs can be challenged with RFC 2617 digest authentication. The backend supports two ways to configure credentials, and they can be combined.

### Static `auth_users` dict

Pass a `username → password` mapping at construction. Every INVITE without valid credentials gets a 401 challenge; on retry, the response is validated against the dict.

```python
backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
    auth_users={"6001": "secret123", "agent": "s3cret"},
    auth_realm="mycompany.com",   # appears in the WWW-Authenticate header
)
```

Best for single-tenant deployments where the credential set is small and known at startup.

### Runtime resolver — `set_auth_resolver()`

For multi-tenant or large credential stores, install a callback that looks up the password on demand. The resolver is consulted on every authenticated INVITE, so credentials can be added, rotated, or revoked without restarting the backend.

```python
def lookup_password(username: str) -> str | None:
    """Look up the password for a SIP username — None denies the call."""
    row = inbound_credentials_cache.get(username)  # in-memory cache populated from DB
    return row["password"] if row else None

backend.set_auth_resolver(lookup_password)
```

Key properties:

- **Resolver wins over the static dict.** When both are configured, the resolver is consulted first; the dict is only used as fallback when the resolver returns `None`.
- **Synchronous.** The resolver runs inside the SIP message loop, so back it with an in-memory cache when the source of truth is remote (database, secret manager). Refresh the cache on credential changes.
- **Exceptions are caught.** A raising resolver is treated as denial — the call is rejected with 403, not propagated up. A buggy lookup callback can't crash the SIP message loop.
- **`set_auth_resolver(None)`** removes a previously installed resolver and falls back to the dict (or disables auth entirely if `auth_users` is also unset).

Pair the resolver with `set_auth_resolver(None)` + `auth_users=None` to dynamically enable/disable auth at runtime.

### Multi-tenant pattern

In a SaaS deployment where each tenant has their own SIP trunk credentials, the resolver becomes the routing primitive — once a username authenticates, it tells you which tenant owns the call:

```python
# At startup: build a per-process credential cache from the DB.
credentials: dict[str, str] = {}     # username → password
tenant_by_username: dict[str, str] = {}  # username → tenant_id

async def refresh_credentials() -> None:
    rows = await db.fetch(
        "SELECT username, password, tenant_id FROM sip_trunks WHERE auth_enabled"
    )
    credentials.clear()
    tenant_by_username.clear()
    for row in rows:
        credentials[row["username"]] = row["password"]
        tenant_by_username[row["username"]] = row["tenant_id"]

await refresh_credentials()
backend.set_auth_resolver(lambda u: credentials.get(u))

# Re-call refresh_credentials() after any DB write that changes credentials —
# the next INVITE picks up the new state.

@backend.on_call
async def handle_call(session):
    # Once authenticated, session.metadata["caller_user"] is the username
    # that just satisfied the digest challenge — look up the tenant.
    username = session.metadata.get("caller_user")
    tenant_id = tenant_by_username.get(username)
    await route_to_tenant(tenant_id, session)
```

This pattern lets the application own credential storage entirely — no need to hold every tenant's secrets in `SIPVoiceBackend`'s constructor argument, and no restart required when a tenant onboards.

### Empty-dict gotcha (fixed in this release)

If you want to start with no credentials and add them later (via mutation or a resolver), pass `auth_users=None` (the default) — not `auth_users={}`. The auth gate (`backend.has_auth()`) returns `False` for both `None` and `{}` until a credential source is actually populated. This guards against accidentally enabling auth challenges before any credentials exist.

### Realm

`auth_realm` (default `"roomkit"`) is the value sent in the `WWW-Authenticate` header. Most carriers don't care what it says — they sign the digest with whatever realm they receive — so a single global realm is usually fine. Use a per-deployment realm only when your carrier requires it.

## Pre-accept rejection — `set_invite_filter()`

By default the SIP backend auto-accepts any INVITE that passes auth, sends `200 OK`, and then fires the `on_call` callback. Applications that want to reject calls based on routing rules (DID not provisioned, tenant not authorized, outside business hours, etc.) can do so from `on_call` by calling `backend.disconnect(session)` — but the carrier will already have seen `200 OK` and the call appears in CDRs as briefly answered.

`set_invite_filter` installs a hook that runs inside `_handle_invite` *before* `200 OK`. The filter receives the `IncomingCall`, returns `None` to accept (proceed to SDP and `200 OK`) or a `(status, reason)` tuple to reject with that 4xx/5xx response. The carrier never sees `200 OK` on a rejection.

```python
async def my_invite_filter(call) -> tuple[int, str] | None:
    """Accept calls only for provisioned DIDs."""
    callee = call.callee  # e.g. "sip:8888@my.host"
    did = callee.split("@", 1)[0].split(":", 1)[-1]
    route = await db.find_did_route(did)
    if route is None:
        return (404, f"Number {did} Not Found")
    if route.tenant_disabled:
        return (403, "Forbidden")
    return None  # accept

backend.set_invite_filter(my_invite_filter)
```

Key properties:

- **Runs after auth.** A filter receiving the call can trust that any digest authentication has already succeeded. The authenticated SIP username is available via `call.invite.get_header("Authorization")` (parse with `aiosipua.parse_auth`) or, less safely, via `call.invite.from_addr.uri.user`.
- **Sync or async.** The dispatcher detects coroutine functions and awaits them. Async filters run inside the SIP message dispatch task — keep DB / network calls fast (the dialog is half-set-up while the filter runs).
- **Exception-safe.** A raising filter is caught and treated as a `500 Server Internal Error` rejection. A buggy lookup callback can't crash the SIP message loop or affect other in-flight INVITEs.
- **Choose appropriate status codes** for the rejection: `403 Forbidden` (no permission), `404 Not Found` (no route), `488 Not Acceptable Here` (codec/SDP), `486 Busy Here`, etc. The reason phrase appears in the `SIP/2.0 4xx <reason>` line so keep it generic — carrier and CDR fields can see it. Internal identifiers (tenant UUIDs, agent IDs) should not appear in the reason text.
- **`set_invite_filter(None)`** removes a previously installed filter and reverts to the default auto-accept behavior.

## Callbacks

The SIP backend provides two additional callbacks beyond the standard `VoiceBackend` interface:

### `on_call(callback)`

Fired after an incoming INVITE is accepted and the RTP session is active. This is where you route the session to a room:

```python
async def handle_call(session: VoiceSession):
    room_id = session.metadata.get("room_id", session.id)
    await kit.create_room(room_id=room_id)
    await kit.attach_channel(room_id, "voice")
    # Push model: pass the SIP-created session to join()
    await kit.join(room_id, "voice", session=session)

backend.on_call(handle_call)
```

### `on_call_disconnected(callback)`

Fired when the remote party sends BYE:

```python
async def handle_disconnect(session: VoiceSession):
    # Previously disconnect_voice() + close_room(), now unified as leave()
    await kit.leave(session)
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

Unlike other backends where you call `kit.join()` (pull model) to create a session, SIP sessions are created automatically during INVITE handling. Use `kit.join()` with the push model to bind the pre-created session:

```python
# In your on_call handler:
await kit.join(room_id, "voice", session=session)
```

## Disconnecting

Call `backend.disconnect(session)` to hang up from the server side. This sends a SIP BYE to the remote party and closes the RTP session:

```python
await backend.disconnect(session)
# SIP BYE is sent, RTP session is closed, session state → ENDED
```

## DTMF

### Inbound (receiving)

DTMF works the same as the RTP backend — digits arrive out-of-band via RFC 4733 and integrate with the hook system:

```python
@kit.hook(HookTrigger.ON_DTMF, execution=HookExecution.ASYNC)
async def on_dtmf(event, ctx):
    print(f"DTMF digit: {event.digit}, duration: {event.duration_ms}ms")
```

### Outbound (sending)

You can send DTMF digits into an active call via `VoiceChannel.send_dtmf()`. This is essential for AI agents navigating IVR menus, entering PINs, or interacting with phone systems:

```python
# Send a single digit
voice.send_dtmf(session, "1")

# Send with custom duration (ms)
voice.send_dtmf(session, "#", duration_ms=250)

# Valid digits: 0-9, *, #, A-D
```

Digits are sent as RFC 4733 telephone-events (out-of-band). See the `examples/voice_sip_dtmf.py` example for a complete AI agent that navigates an IVR menu using tool calling.

## Capabilities

| Capability | Description |
|---|---|
| `DTMF_SIGNALING` | DTMF digits sent and received out-of-band via RFC 4733. |
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

## Jitter buffer tuning

The SIP backend uses a packet-level jitter buffer in the RTP bridge to smooth out network timing variations. The defaults are tuned for low-latency voice AI (start playout immediately, tolerate small jitter), but you can adjust them for different network conditions:

```python
# Lossy / high-jitter network — larger buffer, pre-fill before playout
backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
    jitter_capacity=64,     # ~1.3 s buffer
    jitter_prefetch=4,      # wait for 4 packets (~80 ms) before playout
    skip_audio_gaps=False,  # fill gaps with silence for continuous playout
)

# Ultra-low latency (LAN / localhost)
backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="10.0.0.5",
    rtp_port_start=10000,
    jitter_capacity=8,   # minimal buffer
    jitter_prefetch=0,   # start immediately
)
```

| Parameter | Effect of increasing | Trade-off |
|---|---|---|
| `jitter_capacity` | Absorbs larger bursts of delayed packets | Higher memory usage; stale packets stay buffered longer |
| `jitter_prefetch` | Smoother playout start, fewer underruns | Adds fixed latency before audio begins |
| `skip_audio_gaps` (off) | Continuous audio stream with silence fill | May mask packet loss from downstream processing |

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
