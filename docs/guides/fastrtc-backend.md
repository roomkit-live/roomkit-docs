# FastRTC Voice Backend

RoomKit provides two FastRTC-based voice backends for browser-to-server real-time audio. Both use RoomKit’s vendored headless FastRTC transport; Gradio is not required.

| Backend | Transport | Use case | VAD |
|---------|-----------|----------|-----|
| `FastRTCVoiceBackend` | WebSocket | Traditional STT/TTS pipeline | Client-side (pipeline) |
| `FastRTCRealtimeTransport` | WebRTC | Speech-to-speech AI (Gemini Live, OpenAI Realtime) | Server-side (provider) |

## Installation

```bash
pip install roomkit[fastrtc] fastapi uvicorn
```

This installs the WebRTC and audio dependencies, including aiortc and numpy.

## FastRTCVoiceBackend (WebSocket)

The traditional voice pipeline path. Audio flows through VAD, STT, AI, and TTS stages on the server.

### Architecture

```
Browser mic → WebSocket → FastRTCVoiceBackend
  → AudioPipeline: [Resampler] → [AEC] → [Denoiser] → VAD
  → STT (Deepgram, sherpa-onnx, etc.)
  → AI (Claude, GPT, etc.)
  → TTS (ElevenLabs, etc.)
  → mu-law encode → WebSocket → Browser speaker
```

### Quick start

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

from roomkit import RoomKit, VoiceChannel, AIChannel, ChannelCategory
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig
from roomkit.voice.backends.fastrtc import FastRTCVoiceBackend, mount_fastrtc_voice
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.vad.energy import EnergyVADProvider
from roomkit.voice.stt.deepgram import DeepgramConfig, DeepgramSTTProvider
from roomkit.voice.tts.elevenlabs import ElevenLabsConfig, ElevenLabsTTSProvider

kit = RoomKit()

backend = FastRTCVoiceBackend(
    input_sample_rate=48000,   # Browser mic rate
    output_sample_rate=24000,  # TTS output rate
)

vad = EnergyVADProvider(energy_threshold=300.0, silence_threshold_ms=600)
pipeline = AudioPipelineConfig(vad=vad)
stt = DeepgramSTTProvider(config=DeepgramConfig(api_key="...", model="nova-3"))
tts = ElevenLabsTTSProvider(config=ElevenLabsConfig(api_key="..."))

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend, pipeline=pipeline)
kit.register_channel(voice)

ai = AIChannel(
    "ai",
    provider=AnthropicAIProvider(
        AnthropicConfig(api_key="...", model="claude-opus-5")
    ),
)
kit.register_channel(ai)


async def session_factory(websocket_id: str):
    """Auto-create room + session when a browser connects."""
    room = await kit.create_room()
    await kit.attach_channel(room.id, "voice")
    await kit.attach_channel(room.id, "ai", category=ChannelCategory.INTELLIGENCE)
    # Pull model: kit.join() creates the session, binds it, and wires recording
    session = await kit.join(room.id, "voice", participant_id="browser-user")
    session.metadata["websocket_id"] = websocket_id
    return session


@asynccontextmanager
async def lifespan(app: FastAPI):
    mount_fastrtc_voice(app, backend, path="/voice", session_factory=session_factory)
    yield
    await kit.close()


app = FastAPI(lifespan=lifespan)
```

### Constructor parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_sample_rate` | `int` | `48000` | Browser microphone sample rate. FastRTC defaults to 48kHz. |
| `output_sample_rate` | `int` | `24000` | TTS output sample rate. Must match your TTS provider's native rate. |
| `audio_queue_maxsize` | `int` | `1000` | Per-session outbound audio queue depth. |

### mount_fastrtc_voice parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `app` | `FastAPI` | required | FastAPI application instance. |
| `backend` | `FastRTCVoiceBackend` | required | The backend instance. |
| `path` | `str` | `"/fastrtc"` | Base path for the FastRTC endpoints. |
| `session_factory` | `async (str) → VoiceSession` | `None` | Called with `websocket_id` when a client connects. If not provided, sessions must be created manually before clients connect. |
| `auth` | `async (WebSocket) → dict \| None` | `None` | Authentication callback. Return a metadata dict to accept, `None` to reject. |

### Endpoints created

When mounted at `/voice`, FastRTC creates:

- **`/voice/websocket/offer`** — WebSocket endpoint for audio streaming
- **`/voice/webrtc/offer`** — WebRTC offer endpoint (POST)

### Session lifecycle

```
1. Browser connects to /voice/websocket/offer
2. Browser sends: {"event": "start", "websocket_id": "abc123"}
3. session_factory("abc123") → creates Room + VoiceSession
4. Backend registers WebSocket → fires on_session_ready callbacks
5. Audio frames flow: browser → backend → pipeline → STT → AI → TTS → browser
6. Browser disconnects → session cleanup
```

## FastRTCRealtimeTransport (WebRTC)

The speech-to-speech path. Audio passes through to the AI provider (Gemini Live, OpenAI Realtime) which handles VAD and response generation server-side.

### Architecture

```
Browser mic → WebRTC → FastRTCRealtimeTransport
  → Raw PCM bytes → Provider (Gemini Live / OpenAI Realtime)
  → Provider generates audio + transcriptions
  → PCM playback FIFO → Opus encode → RTP audio track → Browser speaker
```

RoomKit 0.75 sends realtime audio on the remote WebRTC track by default,
with Opus preferred during SDP negotiation. The DataChannel carries controls
and transcriptions. Playback has one bounded PCM FIFO, a 40 ms startup reserve
and 20 ms frames; interruption clears queued PCM before the next frame is read.
A short fade softens starts, stops and starvation without altering every chunk.
An underrun is logged when provider audio runs out during a response.

For clients released before RTP playback support, use
`FastRTCRealtimeTransport(audio_transport="datachannel")`. On a shared transport,
pass `session_metadata={"audio_transport": "datachannel"}` to
`channel.start_session(...)` for those clients. Negotiate this in your app’s
session-start API before submitting the WebRTC offer. A client consumes either
RTP audio or legacy media messages for that session.

### Quick start

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

from roomkit import RoomKit, RealtimeVoiceChannel
from roomkit.providers.gemini.realtime import GeminiLiveProvider
from roomkit.voice.realtime.fastrtc_transport import (
    FastRTCRealtimeTransport,
    mount_fastrtc_realtime,
)

kit = RoomKit()

provider = GeminiLiveProvider(
    api_key="...",
    model="gemini-3.8-live",
)

transport = FastRTCRealtimeTransport(
    input_sample_rate=16000,
    output_sample_rate=24000,
)

channel = RealtimeVoiceChannel(
    "realtime-voice",
    provider=provider,
    transport=transport,
    system_prompt="You are a helpful assistant.",
    voice="Aoede",
)
kit.register_channel(channel)


async def on_client_connected(webrtc_id: str) -> None:
    room = await kit.create_room()
    await kit.attach_channel(room.id, "realtime-voice")
    await channel.start_session(room.id, "user-1", connection=webrtc_id)


transport.on_client_connected(on_client_connected)


@asynccontextmanager
async def lifespan(app: FastAPI):
    mount_fastrtc_realtime(app, transport, path="/rtc-realtime")
    yield
    await kit.close()


app = FastAPI(lifespan=lifespan)
```

### Constructor parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_sample_rate` | `int` | `16000` | Browser microphone sample rate. |
| `output_sample_rate` | `int` | `24000` | Provider output sample rate. |
| `audio_transport` | `"webrtc" \| "datachannel"` | `"webrtc"` | RTP audio (Opus preferred), or legacy mu-law media messages. |

### mount_fastrtc_realtime parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `app` | `FastAPI` | required | FastAPI application instance. |
| `transport` | `FastRTCRealtimeTransport` | required | The transport instance. |
| `path` | `str` | `"/rtc-realtime"` | Base path for WebRTC endpoints. |
| `auth` | `async (context) → dict \| None` | `None` | Authentication callback. Receives the FastRTC context (with `webrtc_id`, connection info). Return metadata dict to accept, `None` to reject. |
| `rtc_configuration` | `dict \| callable \| None` | `None` | Server ICE configuration, or a zero-argument sync/async callback returning it for each new peer. |
| `concurrency_limit` | `int \| None` | `None` | Maximum simultaneous connections; `None` uses the stream default of 1. |

### Short-lived TURN credentials

Pass a callback when TURN credentials expire. Mounting performs no credential
lookup; each new offer resolves the callback before creating its server peer.
Keep any shared cache in the callback or credential provider and refresh it before
credentials expire. Existing peers keep their own configuration.

```python
import httpx

async def rtc_configuration():
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get("https://credentials.example/ice-servers")
        response.raise_for_status()
        return response.json()  # {"iceServers": [...]}

mount_fastrtc_realtime(app, transport, rtc_configuration=rtc_configuration)
```

If resolution fails, the offer returns
`{"status": "failed", "meta": {"error": "rtc_configuration_failed"}}`.
No peer or connection slot is allocated. Display a retryable connection error;
a subsequent offer can succeed on the same route when the provider recovers.
Exceptions and credential details are not included in this response.

The underlying `Stream(server_rtc_configuration=...)` accepts the same callbacks.
Static dictionaries remain supported; `None` uses aiortc defaults, while
`{"iceServers": []}` disables server STUN/TURN. See
[`examples/realtime_voice_fastrtc.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/realtime_voice_fastrtc.py)
for a runnable application using `TURN_CREDENTIALS_URL`.

### Endpoints created

When mounted at `/rtc-realtime`:

- **`/rtc-realtime/webrtc/offer`** — WebRTC SDP offer endpoint (POST)

### Connection flow

```
1. Browser creates RTCPeerConnection
2. Browser creates DataChannel named "text" (required)
3. Browser sends SDP offer to /rtc-realtime/webrtc/offer
4. FastRTC negotiates ICE, DTLS, SRTP
5. handler.start_up() → transport registers handler → fires on_client_connected
6. App calls channel.start_session(room, participant, connection=webrtc_id)
7. Audio flows bidirectionally via WebRTC media tracks
8. Transcriptions sent via DataChannel as JSON
```

## Audio format

`FastRTCRealtimeTransport` uses the negotiated WebRTC codec (Opus preferred)
for audio in both directions. The provider interface remains PCM16; RoomKit
resamples when the provider and transport PCM rates differ. Browser and native
WebRTC implementations handle packet jitter and decoding.

The legacy DataChannel option and `FastRTCVoiceBackend` WebSocket protocol
retain mu-law, base64-wrapped as
`{"event": "media", "media": {"payload": "..."}}`. This is separate from the
generic `WebSocketRealtimeTransport`, whose default is binary PCM16 as of 0.75;
set `audio_format="base64_json"` there for existing JSON clients. SIP continues
to prefer G.722 with negotiated G.711 fallbacks.

## Browser client

### WebSocket connection (JavaScript)

```javascript
const ws = new WebSocket('ws://localhost:8000/voice/websocket/offer');
const wsId = crypto.randomUUID();

ws.onopen = () => {
  ws.send(JSON.stringify({ event: 'start', websocket_id: wsId }));

  // Capture mic and send mu-law audio
  navigator.mediaDevices.getUserMedia({ audio: true }).then(stream => {
    const ctx = new AudioContext({ sampleRate: 48000 });
    const source = ctx.createMediaStreamSource(stream);
    const processor = ctx.createScriptProcessor(4096, 1, 1);

    processor.onaudioprocess = (e) => {
      const float32 = e.inputBuffer.getChannelData(0);
      const mulaw = encodeMulaw(float32);  // PCM float → mu-law bytes
      const b64 = btoa(String.fromCharCode(...mulaw));
      ws.send(JSON.stringify({ event: 'media', media: { payload: b64 } }));
    };

    source.connect(processor);
    processor.connect(ctx.destination);
  });
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  // Play received audio
  if (data.event === 'media') {
    const mulaw = Uint8Array.from(atob(data.media.payload), c => c.charCodeAt(0));
    // Decode mu-law → PCM → play via AudioContext
  }

  // Display transcriptions
  if (data.type === 'transcription') {
    console.log(`${data.data.role}: ${data.data.text}`);
  }
};
```

### WebRTC connection (JavaScript)

```javascript
const pc = new RTCPeerConnection();
const webrtcId = crypto.randomUUID();

// Mic → peer connection
const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// Data channel for transcriptions (must be created before offer)
const dc = pc.createDataChannel('text');
dc.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'transcription') {
    console.log(`${data.data.role}: ${data.data.text}`);
  }
};

// Remote audio
pc.ontrack = (event) => {
  const audio = new Audio();
  audio.srcObject = event.streams[0] ?? new MediaStream([event.track]);
  audio.play();
};

// ICE candidates
pc.onicecandidate = ({ candidate }) => {
  if (candidate) {
    fetch('/rtc-realtime/webrtc/offer', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        candidate: candidate.toJSON(),
        webrtc_id: webrtcId,
        type: 'ice-candidate',
      }),
    });
  }
};

// Create and send offer
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

const resp = await fetch('/rtc-realtime/webrtc/offer', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ sdp: offer.sdp, type: offer.type, webrtc_id: webrtcId }),
});
const answer = await resp.json();
await pc.setRemoteDescription(answer);
```

!!! tip "Ready-made client"
    See [`examples/fastrtc_client.html`](https://github.com/roomkit-live/roomkit/blob/main/examples/fastrtc_client.html) for a complete browser client that supports both WebSocket and WebRTC modes with mu-law encoding/decoding, VU meter, and transcription display.

## Authentication

Both backends support pluggable authentication via an `auth` callback.

### WebSocket auth

```python
async def authenticate(websocket) -> dict[str, object] | None:
    """Validate token from query string or headers."""
    token = websocket.query_params.get("token")
    if not token or not await verify_token(token):
        return None  # Reject connection
    return {"user_id": "...", "role": "agent"}  # Accept

mount_fastrtc_voice(app, backend, path="/voice", auth=authenticate,
                    session_factory=session_factory)
```

Auth metadata is available inside `session_factory` via the `auth_context` context variable:

```python
from roomkit.voice.auth import auth_context

async def session_factory(websocket_id: str):
    meta = auth_context.get()  # {"user_id": "...", "role": "agent"}
    room = await kit.create_room()
    # ... use meta to customize session
```

### WebRTC auth

```python
async def authenticate(ctx) -> dict[str, object] | None:
    """Validate from FastRTC connection context."""
    # ctx has webrtc_id and connection metadata
    return {"authenticated": True}

mount_fastrtc_realtime(app, transport, path="/rtc-realtime", auth=authenticate)
```

## Comparing the two backends

| Aspect | FastRTCVoiceBackend | FastRTCRealtimeTransport |
|--------|---------------------|--------------------------|
| Transport | WebSocket | WebRTC (ICE, DTLS, SRTP) |
| Audio codec | mu-law (both directions) | PCM inbound, mu-law outbound |
| VAD | Pipeline-side (EnergyVAD, SherpaOnnx, etc.) | Provider-side (Gemini/OpenAI) |
| STT/TTS | Separate providers (Deepgram + ElevenLabs) | Built into the provider |
| Channel type | `VoiceChannel` | `RealtimeVoiceChannel` |
| Pipeline stages | Full (AEC, AGC, denoiser, VAD, diarization) | None (passthrough) |
| Latency | ~100-200ms (STT + AI + TTS) | ~50-150ms (speech-to-speech) |
| NAT traversal | N/A (WebSocket) | Full ICE with STUN/TURN |
| Use case | Custom STT/TTS stack, pipeline control | Low-latency speech-to-speech AI |

## Capabilities

`FastRTCVoiceBackend` declares `VoiceCapability.NONE` — all intelligence is delegated to the AudioPipeline.

`FastRTCRealtimeTransport` is a passthrough — the provider handles speech detection, so no pipeline capabilities are needed.

## Pipeline integration

The `FastRTCVoiceBackend` integrates with the full AudioPipeline. You can add any combination of pipeline stages:

```python
from roomkit.voice.pipeline import AudioPipelineConfig, WavFileRecorder, RecordingConfig
from roomkit.voice.pipeline.aec.webrtc import WebRTCAECProvider
from roomkit.voice.pipeline.denoiser.rnnoise import RNNoiseDenoiserProvider
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADProvider, SherpaOnnxVADConfig

pipeline = AudioPipelineConfig(
    vad=SherpaOnnxVADProvider(SherpaOnnxVADConfig(model="path/to/model.onnx")),
    aec=WebRTCAECProvider(sample_rate=16000),
    denoiser=RNNoiseDenoiserProvider(sample_rate=16000),
    recorder=WavFileRecorder(),
    recording_config=RecordingConfig(
        storage="./recordings",
        storage_encrypted_at_rest=True,
    ),
)

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend, pipeline=pipeline)
```

The pipeline processes inbound audio in this order:

```
[Resampler] → [Recorder tap] → [AEC] → [AGC] → [Denoiser] → VAD → [Diarization] + [DTMF]
```

## Examples

- [`examples/voice_fastrtc.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_fastrtc.py) — FastRTCVoiceBackend with Deepgram STT + Claude + ElevenLabs TTS (includes inline browser client)
- [`examples/realtime_voice_fastrtc.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/realtime_voice_fastrtc.py) — FastRTCRealtimeTransport with Gemini Live
- [`examples/fastrtc_client.html`](https://github.com/roomkit-live/roomkit/blob/main/examples/fastrtc_client.html) — Standalone browser client supporting both modes

## API Reference

- ::: roomkit.voice.backends.fastrtc.FastRTCVoiceBackend
- ::: roomkit.voice.realtime.fastrtc_transport.FastRTCRealtimeTransport

See [Voice Channel API](../api/providers-voice.md) and [Realtime Voice API](../api/providers-realtime-voice.md) for channel-level documentation.
