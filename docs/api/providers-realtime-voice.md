# Realtime Voice

Speech-to-speech AI conversations using providers like Google Gemini Live, OpenAI Realtime, and xAI Grok Realtime. Audio flows directly between the client and the AI provider; transcriptions are emitted into the Room so other channels (supervisor dashboards, logging) see the conversation.

## RealtimeVoiceChannel

::: roomkit.RealtimeVoiceChannel

## ToolHandler

```python
ToolHandler = Callable[
    [str, dict[str, Any]],
    Awaitable[str],
]
```

Async callback for executing tool/function calls from the AI provider. Receives `(tool_name, arguments)` and returns a JSON string.

## Provider ABC

::: roomkit.voice.realtime.provider.RealtimeVoiceProvider

## Transport ABC

All realtime transports extend [`VoiceBackend`](providers-voice.md#roomkit.voice.backends.base.VoiceBackend).

## Session & State

See [`VoiceSession`](providers-voice.md#roomkit.voice.base.VoiceSession) and [`VoiceSessionState`](providers-voice.md#roomkit.voice.base.VoiceSessionState).

## Events

::: roomkit.voice.realtime.events.RealtimeTranscriptionEvent

::: roomkit.voice.realtime.events.RealtimeSpeechEvent

::: roomkit.voice.realtime.events.RealtimeToolCallEvent

::: roomkit.voice.realtime.events.RealtimeErrorEvent

## Callback Types

| Callback | Signature |
|----------|-----------|
| `RealtimeAudioCallback` | `(RealtimeSession, bytes) -> Any` |
| `RealtimeTranscriptionCallback` | `(RealtimeSession, str, str, bool) -> Any` |
| `RealtimeSpeechStartCallback` | `(RealtimeSession) -> Any` |
| `RealtimeSpeechEndCallback` | `(RealtimeSession) -> Any` |
| `RealtimeToolCallCallback` | `(RealtimeSession, str, str, dict) -> Any` |
| `RealtimeResponseStartCallback` | `(RealtimeSession) -> Any` |
| `RealtimeResponseEndCallback` | `(RealtimeSession) -> Any` |
| `RealtimeErrorCallback` | `(RealtimeSession, str, str) -> Any` |

## Concrete Providers

### Gemini Live

```python
from roomkit.providers.gemini.realtime import GeminiLiveProvider

provider = GeminiLiveProvider(
    api_key="...",
    model="gemini-2.5-flash-native-audio-preview-12-2025",
)
```

Install with: `pip install roomkit[realtime-gemini]`

Supports `provider_config` metadata keys: `language`, `top_p`, `top_k`, `max_output_tokens`, `seed`, `enable_affective_dialog`, `thinking_budget`, `proactive_audio`, `start_of_speech_sensitivity`, `end_of_speech_sensitivity`, `silence_duration_ms`, `prefix_padding_ms`, `no_interruption`.

### OpenAI Realtime

```python
from roomkit.providers.openai.realtime import OpenAIRealtimeProvider

provider = OpenAIRealtimeProvider(
    api_key="...",
    model="gpt-4o-realtime-preview",
)
```

Install with: `pip install roomkit[realtime-openai]`

### xAI Grok Realtime

```python
from roomkit.providers.xai.config import XAIRealtimeConfig
from roomkit.providers.xai.realtime import XAIRealtimeProvider

provider = XAIRealtimeProvider(
    XAIRealtimeConfig(
        api_key="...",
        model="grok-3-fast",
        voice="eve",
        transcription_model="grok-2-audio",
    )
)
```

Install with: `pip install websockets`

Supports `provider_config` metadata keys: `turn_detection_type`, `threshold`, `silence_duration_ms`, `prefix_padding_ms`, `transcription_model`, `input_audio_format`, `output_audio_format`, `modalities`.

Native tools: `web_search`, `x_search` (passed as `{"type": "web_search"}`).

Available voices: eve, ara, rex, sal, leo.

## Transports

### WebSocket Transport

```python
from roomkit.voice.realtime.ws_transport import WebSocketRealtimeTransport

transport = WebSocketRealtimeTransport()
```

Handles bidirectional audio over WebSocket. Client sends `{"type": "audio", "data": "<base64 PCM>"}`, server sends audio, transcriptions, and speaking indicators as JSON messages.

Install with: `pip install roomkit[websocket]`

### FastRTC Transport (WebRTC)

```python
from roomkit.voice.realtime.fastrtc_transport import (
    FastRTCRealtimeTransport,
    mount_fastrtc_realtime,
)

transport = FastRTCRealtimeTransport(
    input_sample_rate=16000,   # Mic input sample rate
    output_sample_rate=24000,  # Provider output sample rate
)

# Mount WebRTC endpoints on FastAPI app
mount_fastrtc_realtime(app, transport, path="/rtc-realtime")
```

WebRTC-based transport using [FastRTC](https://github.com/gradio-app/fastrtc) in **passthrough mode** (no VAD). Audio flows bidirectionally between the browser and the speech-to-speech AI provider, which handles its own server-side VAD.

Unlike the `FastRTCVoiceBackend` (which uses `ReplyOnPause` for VAD in the traditional `VoiceChannel` pipeline), this transport simply passes audio through — ideal for speech-to-speech providers like OpenAI Realtime or Gemini Live.

Install with: `pip install roomkit[fastrtc]`

**Connection flow:**

1. `mount_fastrtc_realtime(app, transport)` creates a FastRTC `Stream` with a passthrough handler
2. Browser connects via WebRTC → FastRTC calls `handler.copy()` → `start_up()`
3. `start_up()` reads `webrtc_id` from FastRTC context, registers with transport
4. Transport fires `on_client_connected` callback (if set) — use to auto-create sessions
5. App calls `channel.start_session(room_id, participant_id, connection=webrtc_id)`
6. `start_session()` → `transport.accept(session, webrtc_id)` maps session to handler
7. `receive()`/`emit()` now flow audio with session context

**Audio format conversion:**

FastRTC works with `(sample_rate, numpy.ndarray[int16])` tuples. The transport ABC uses raw `bytes` (PCM16 LE). Conversion is automatic:

- Inbound: `audio_array.tobytes()` → fire callbacks with bytes
- Outbound: `np.frombuffer(audio_bytes, dtype=np.int16)` → return as `(rate, array)`

**DataChannel messages:**

JSON messages (transcriptions, speaking indicators) are sent via the WebRTC DataChannel using `send_message()`.

| Transport | Protocol | VAD | Dependency |
|-----------|----------|-----|------------|
| `WebSocketRealtimeTransport` | WebSocket | Provider-side | `roomkit[websocket]` |
| `FastRTCRealtimeTransport` | WebRTC | Provider-side | `roomkit[fastrtc]` |

Lazy-loaded via `roomkit.voice.get_fastrtc_realtime_transport()` and `roomkit.voice.get_mount_fastrtc_realtime()` to avoid requiring `fastrtc`/`numpy` at import time.

## Mock Classes

::: roomkit.voice.realtime.mock.MockRealtimeProvider

::: roomkit.voice.realtime.mock.MockRealtimeTransport

## Usage Example

```python
from roomkit import RoomKit, RealtimeVoiceChannel
from roomkit.providers.gemini.realtime import GeminiLiveProvider
from roomkit.voice.realtime.ws_transport import WebSocketRealtimeTransport

# Configure provider and transport
provider = GeminiLiveProvider(api_key="...")
transport = WebSocketRealtimeTransport()

# Create and register the channel
channel = RealtimeVoiceChannel(
    "realtime-voice",
    provider=provider,
    transport=transport,
    system_prompt="You are a helpful voice assistant.",
    voice="Aoede",
)

kit = RoomKit()
kit.register_channel(channel)

# Create room and attach
room = await kit.create_room(room_id="voice-room")
await kit.attach_channel("voice-room", "realtime-voice")

# When a WebSocket connection arrives:
session = await channel.start_session(
    "voice-room", "participant-1", websocket
)

# Transcriptions are emitted as RoomEvents —
# other channels in the room see the conversation.

# Text from other channels is injected into the AI session:
# supervisor types "Offer 20% discount" via WebSocket →
# AI incorporates it into its next spoken response.
```

### With Tool Calling

```python
import json

from roomkit import Tool

async def lookup_order(order_id: str) -> str:
    return json.dumps({"status": "shipped", "eta": "Tomorrow"})

order_tool = Tool(
    name="lookup_order",
    description="Look up an order by ID",
    parameters={
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
    handler=lookup_order,
)

channel = RealtimeVoiceChannel(
    "realtime-voice",
    provider=provider,
    transport=transport,
    system_prompt="You are a helpful assistant with access to tools.",
    tools=[order_tool],
)
```

### With MCP (Advanced)

For MCP integration, use `tool_handler` to route calls to the MCP server:

```python
import json

from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

# Connect to MCP server
read, write, _ = await streamablehttp_client("http://localhost:9998/mcp")
mcp = ClientSession(read, write)
await mcp.initialize()

# Discover tools
tools_result = await mcp.list_tools()
tools = [
    {
        "name": t.name,
        "description": t.description or t.name,
        "parameters": t.inputSchema or {"type": "object", "properties": {}},
    }
    for t in tools_result.tools
]

# Tool handler routes calls to MCP
async def handle_tool(name, arguments):
    result = await mcp.call_tool(name, arguments)
    return json.dumps({"result": "\n".join(b.text for b in result.content if hasattr(b, "text"))})

channel = RealtimeVoiceChannel(
    "realtime-voice",
    provider=provider,
    transport=transport,
    system_prompt="You are a helpful assistant with access to tools.",
    tools=tools,
    tool_handler=handle_tool,
)
```

### With Delegation

Wire the `delegate_task` tool into a voice agent:

```python
from roomkit.tasks import DelegateHandler, setup_realtime_delegation, build_delegate_tool

handler = DelegateHandler(kit, notify="realtime-voice")
tool = build_delegate_tool([("exec-agent", "Runs background tasks")])
setup_realtime_delegation(channel, handler, tool=tool)
```

See [`setup_realtime_delegation()`](delegation.md) for details.

### With Vision Injection

Wire screen/camera vision into the voice session:

```python
from roomkit import setup_realtime_vision

setup_realtime_vision(kit, room_id="room-1", voice_channel_id="realtime-voice")
```

Vision descriptions are injected via `inject_text(silent=True)` — the agent sees the context on its next turn without interrupting the conversation. Unchanged descriptions are automatically deduplicated.

See [`setup_realtime_vision()`](../guides/video-vision.md#realtime-voice-integration) for details.

### With FastRTC (WebRTC)

```python
from fastapi import FastAPI
from roomkit import RoomKit, RealtimeVoiceChannel
from roomkit.providers.gemini.realtime import GeminiLiveProvider
from roomkit.voice.realtime.fastrtc_transport import (
    FastRTCRealtimeTransport,
    mount_fastrtc_realtime,
)

app = FastAPI()

provider = GeminiLiveProvider(api_key="...")
transport = FastRTCRealtimeTransport(
    input_sample_rate=16000,
    output_sample_rate=24000,
)

channel = RealtimeVoiceChannel(
    "realtime-voice",
    provider=provider,
    transport=transport,
    system_prompt="You are a helpful voice assistant.",
    voice="Aoede",
)

kit = RoomKit()
kit.register_channel(channel)

# Mount WebRTC endpoints
mount_fastrtc_realtime(app, transport, path="/rtc-realtime")

# Auto-create sessions when WebRTC clients connect
async def on_connected(webrtc_id: str):
    room = await kit.create_room()
    await kit.attach_channel(room.id, "realtime-voice")
    await channel.start_session(room.id, "user-1", connection=webrtc_id)

transport.on_client_connected(on_connected)
```
