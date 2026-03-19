# WebTransport Voice Backend

The `WebTransportBackend` uses [WebTransport](https://www.w3.org/TR/webtransport/) (HTTP/3 over QUIC) for low-latency audio transport. Audio is exchanged as **unreliable QUIC datagrams** — similar to UDP, with no head-of-line blocking.

## Why WebTransport?

| Feature | WebRTC | WebTransport |
|---------|--------|--------------|
| Signaling | Complex (ICE/STUN/TURN) | Simple HTTPS URL |
| NAT traversal | Requires STUN/TURN servers | Direct QUIC connection |
| Audio transport | RTP over DTLS-SRTP | QUIC datagrams (unreliable) |
| Head-of-line blocking | No (separate DTLS) | No (datagrams) |
| Server-side | Complex media stack | Standard QUIC server |
| Browser support | All browsers | Chrome, Edge, Firefox (Safari partial) |

WebTransport is a compelling alternative when:

- You want simpler server infrastructure (no STUN/TURN)
- You control both endpoints (no interop with phone networks)
- Low latency matters more than browser compatibility
- You want to avoid WebRTC's complex ICE negotiation

## Installation

```bash
pip install 'roomkit[webtransport]'
```

This installs `aioquic` for QUIC/HTTP3 support.

## TLS Certificate

WebTransport requires TLS. For development, generate a self-signed certificate:

```bash
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
    -days 365 -nodes -keyout key.pem -out cert.pem \
    -subj "/CN=localhost"
```

For production, use a proper certificate from a CA (Let's Encrypt, etc.).

## Basic Usage

```python
from roomkit import RoomKit, VoiceChannel
from roomkit.voice.backends.webtransport import WebTransportBackend

backend = WebTransportBackend(
    host="0.0.0.0",
    port=4433,
    certificate="cert.pem",
    private_key="key.pem",
    input_sample_rate=16000,
    output_sample_rate=16000,
)

voice = VoiceChannel("voice", backend=backend)
kit = RoomKit()
kit.register_channel(voice)

await kit.create_room(room_id="my-room")
await kit.attach_channel("my-room", "voice")
await backend.start()
```

## Wire Protocol

Audio is exchanged as QUIC datagrams with a simple binary format:

```
┌──────────────────┬───────────────────────────┐
│ 2 bytes: uint16  │ N bytes: PCM-16 LE audio  │
│ sample_rate/100  │                           │
│ (little-endian)  │                           │
└──────────────────┴───────────────────────────┘
```

Examples:

- 16000 Hz → header bytes `0xA0 0x00` (160 as LE uint16)
- 48000 Hz → header bytes `0xE0 0x01` (480 as LE uint16)
- 8000 Hz → header bytes `0x50 0x00` (80 as LE uint16)

## Session Factory

When a new WebTransport client connects, the backend needs to create a `VoiceSession`. You can provide a custom factory:

```python
async def session_factory(connection_id: str):
    # Pull model: kit.join() creates the session, binds it, and wires recording
    session = await kit.join(
        "my-room",
        "voice",
        participant_id=f"user-{connection_id}",
    )
    return session

backend.set_session_factory(session_factory)
```

Without a factory, the backend creates sessions with default parameters.

## Callbacks

```python
# Called when audio arrives from a client
backend.on_audio_received(lambda session, frame: ...)

# Called when a new client connects
backend.on_session_ready(lambda session: ...)

# Called when a client disconnects
backend.on_client_disconnected(lambda session: ...)
```

## Browser Client

Connect from a browser using the WebTransport API:

```javascript
const transport = new WebTransport("https://localhost:4433/audio");
await transport.ready;

const writer = transport.datagrams.writable.getWriter();
const reader = transport.datagrams.readable.getReader();

// Send audio: 2-byte header + PCM-16 LE data
function sendAudio(pcmData, sampleRate) {
    const header = new Uint8Array(2);
    const view = new DataView(header.buffer);
    view.setUint16(0, sampleRate / 100, true); // little-endian

    const datagram = new Uint8Array(header.length + pcmData.length);
    datagram.set(header);
    datagram.set(new Uint8Array(pcmData), header.length);
    writer.write(datagram);
}

// Receive audio
async function receiveAudio() {
    while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        const view = new DataView(value.buffer);
        const sampleRate = view.getUint16(0, true) * 100;
        const pcmData = value.slice(2);
        // Play pcmData at sampleRate...
    }
}
```

!!! note "Self-signed certificates"
    For development with self-signed certs, Chrome requires either:

    - Navigate to `chrome://flags/#allow-insecure-localhost`
    - Launch with `--origin-to-force-quic-on=localhost:4433`

## With Audio Bridge

Combine WebTransport with other backends for cross-transport bridging:

```python
from roomkit.voice.backends.webtransport import WebTransportBackend
from roomkit.voice.backends.sip import SIPVoiceBackend

wt_backend = WebTransportBackend(port=4433, certificate="cert.pem", private_key="key.pem")
sip_backend = SIPVoiceBackend(local_sip_addr=("0.0.0.0", 5060))

voice = VoiceChannel("voice", backend=wt_backend, bridge=True)

# Register SIP audio callbacks on the same voice channel
sip_backend.on_audio_received(voice._on_audio_received)
sip_backend.on_session_ready(voice._on_session_ready)
```

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `host` | `"0.0.0.0"` | Bind address for the QUIC server |
| `port` | `4433` | UDP port for the QUIC server |
| `certificate` | `"cert.pem"` | Path to TLS certificate (PEM) |
| `private_key` | `"key.pem"` | Path to TLS private key (PEM) |
| `input_sample_rate` | `16000` | Expected inbound audio sample rate |
| `output_sample_rate` | `16000` | Outbound audio sample rate |
| `path` | `"/audio"` | URL path for WebTransport connections |
| `max_datagram_size` | `65536` | Maximum datagram payload size |

## Lazy Loading

To avoid requiring `aioquic` at import time:

```python
from roomkit.voice import get_webtransport_backend

WebTransportBackend = get_webtransport_backend()
backend = WebTransportBackend(port=4433, certificate="cert.pem", private_key="key.pem")
```
