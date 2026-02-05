# Event-Driven Sources

RoomKit's source system enables **event-driven message ingestion** from persistent connections like WebSockets, NATS, SSE, or custom protocols (e.g., WhatsApp via neonize).

## Overview

Unlike webhook-based providers that receive HTTP POST requests, **SourceProviders** maintain persistent connections and push messages into RoomKit as they arrive.

```python
from roomkit import RoomKit, SourceProvider, SourceStatus, InboundMessage

# Attach an event-driven source
await kit.attach_source("my-channel", my_source)

# Check source health
health = await kit.source_health("my-channel")
print(f"Status: {health.status}, Messages: {health.messages_received}")

# List all sources
sources = kit.list_sources()
# {"my-channel": SourceStatus.CONNECTED, ...}

# Detach when done
await kit.detach_source("my-channel")
```

## Webhook vs Event-Driven

| Aspect | Webhooks | Event Sources |
|--------|----------|---------------|
| Connection | Stateless HTTP | Persistent (WS, TCP, etc.) |
| Initiative | External system pushes | RoomKit subscribes |
| Lifecycle | Per-request | Managed by RoomKit |
| Use cases | Twilio, SendGrid | WebSocket, NATS, neonize |

## Attaching Sources

Use `attach_source()` to connect an event-driven source to a channel:

```python
from roomkit import RoomKit
from my_sources import WebSocketSource

kit = RoomKit()

# Create and attach source
source = WebSocketSource(url="wss://example.com/events")
await kit.attach_source(
    channel_id="websocket-events",
    source=source,
    auto_restart=True,           # Restart on failure (default: True)
    restart_delay=5.0,           # Initial delay between restarts (default: 5.0)
    max_restart_delay=300.0,     # Cap backoff at 5 minutes (default: 300.0)
    max_restart_attempts=10,     # Give up after 10 failures (default: None = unlimited)
    max_concurrent_emits=20,     # Backpressure limit (default: 10)
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `channel_id` | `str` | required | Channel ID for inbound messages |
| `source` | `SourceProvider` | required | The source provider instance |
| `auto_restart` | `bool` | `True` | Auto-restart on unexpected exit |
| `restart_delay` | `float` | `5.0` | Initial delay before first restart |
| `max_restart_delay` | `float` | `300.0` | Maximum delay between restarts (caps exponential backoff) |
| `max_restart_attempts` | `int \| None` | `None` | Max consecutive failures before giving up. `None` = unlimited |
| `max_concurrent_emits` | `int \| None` | `10` | Max concurrent `emit()` calls (backpressure). `None` = unlimited |

### Exponential Backoff

When a source fails and `auto_restart=True`, RoomKit uses exponential backoff:

1. First failure: wait `restart_delay` seconds
2. Second failure: wait `restart_delay * 2` seconds
3. Third failure: wait `restart_delay * 4` seconds
4. ...and so on, capped at `max_restart_delay`

After a successful start, the delay resets to the initial `restart_delay`.

### Backpressure Control

The `max_concurrent_emits` parameter prevents a fast source from overwhelming the system. When the limit is reached, additional `emit()` calls will wait until previous calls complete.

```python
# High-volume source with strict backpressure
await kit.attach_source(
    "firehose",
    high_volume_source,
    max_concurrent_emits=5,  # Only 5 messages processing at once
)

# Low-volume source where backpressure isn't needed
await kit.attach_source(
    "slow-feed",
    slow_source,
    max_concurrent_emits=None,  # No limit
)
```

## Detaching Sources

Stop and remove a source with `detach_source()`:

```python
await kit.detach_source("websocket-events")
```

This will:
1. Call `source.stop()` to signal shutdown
2. Cancel the background task
3. Emit a `source_detached` framework event

## Monitoring Health

Check the health of attached sources:

```python
from roomkit import SourceStatus

# Single source health
health = await kit.source_health("websocket-events")
if health:
    print(f"Status: {health.status}")
    print(f"Connected at: {health.connected_at}")
    print(f"Last message: {health.last_message_at}")
    print(f"Messages received: {health.messages_received}")
    if health.error:
        print(f"Error: {health.error}")

# List all sources
for channel_id, status in kit.list_sources().items():
    print(f"{channel_id}: {status}")
```

### SourceStatus Values

| Status | Description |
|--------|-------------|
| `STOPPED` | Source is not running |
| `CONNECTING` | Establishing connection |
| `CONNECTED` | Active and receiving messages |
| `RECONNECTING` | Connection lost, attempting reconnect |
| `ERROR` | Failed state (check `health.error`) |

## Framework Events

Sources emit framework events for observability:

```python
@kit.on("source_attached")
async def on_attached(event):
    print(f"Source attached: {event.data['source_name']} to {event.channel_id}")

@kit.on("source_detached")
async def on_detached(event):
    print(f"Source detached: {event.data['source_name']} from {event.channel_id}")

@kit.on("source_error")
async def on_error(event):
    print(f"Source error: {event.data['error']} (attempt {event.data['attempt']})")

@kit.on("source_exhausted")
async def on_exhausted(event):
    # Fired when max_restart_attempts is reached
    print(f"Source {event.data['source_name']} gave up after {event.data['attempts']} attempts")
    print(f"Last error: {event.data['last_error']}")
    # Consider alerting, switching to fallback, etc.
```

| Event | Data | Description |
|-------|------|-------------|
| `source_attached` | `source_name` | Source started successfully |
| `source_detached` | `source_name` | Source stopped and removed |
| `source_error` | `source_name`, `error`, `attempt` | Source failed (will retry if `auto_restart=True`) |
| `source_exhausted` | `source_name`, `attempts`, `last_error` | Max restart attempts reached, source gave up |

## Built-in Sources

### WebSocketSource

Connect to a WebSocket server and receive messages:

```python
from roomkit import RoomKit
from roomkit.sources import WebSocketSource

# Basic usage with default JSON parser
source = WebSocketSource(
    url="wss://chat.example.com/events",
    channel_id="websocket-chat",
)
await kit.attach_source("websocket-chat", source)
```

The default parser expects JSON messages with this structure:

```json
{
    "sender_id": "user123",
    "text": "Hello world",
    "external_id": "msg-456",
    "metadata": {"key": "value"}
}
```

#### Custom Message Parser

For non-JSON or custom formats, provide a parser function:

```python
from roomkit import InboundMessage, TextContent

def my_parser(raw: str | bytes) -> InboundMessage | None:
    """Parse custom protocol: SENDER|MESSAGE"""
    if isinstance(raw, bytes):
        raw = raw.decode("utf-8")

    parts = raw.split("|", 1)
    if len(parts) < 2:
        return None  # Skip malformed messages

    return InboundMessage(
        channel_id="custom-ws",
        sender_id=parts[0],
        content=TextContent(body=parts[1]),
    )

source = WebSocketSource(
    url="wss://legacy.example.com/stream",
    channel_id="custom-ws",
    parser=my_parser,
)
```

#### WebSocketSource Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` | `str` | required | WebSocket URL (ws:// or wss://) |
| `channel_id` | `str` | required | Channel ID for emitted messages |
| `parser` | `Callable` | JSON parser | Function to parse raw messages |
| `headers` | `dict[str, str]` | `None` | Additional HTTP headers |
| `subprotocols` | `list[str]` | `None` | WebSocket subprotocols |
| `ping_interval` | `float` | `20.0` | Ping frame interval (seconds) |
| `ping_timeout` | `float` | `20.0` | Pong response timeout (seconds) |
| `close_timeout` | `float` | `10.0` | Close handshake timeout |
| `max_size` | `int` | 1 MB | Maximum message size (bytes) |
| `origin` | `str` | `None` | Origin header value |

#### Bidirectional Communication

WebSocketSource also supports sending messages:

```python
# After attaching and connecting
if source.status == SourceStatus.CONNECTED:
    await source.send('{"type": "ping"}')
    await source.send(b'\x00\x01\x02')  # Binary data
```

#### Installation

WebSocketSource requires the `websockets` package:

```bash
pip install roomkit[websocket]
```

### SSESource

Connect to a Server-Sent Events (SSE) endpoint and receive real-time updates:

```python
from roomkit import RoomKit
from roomkit.sources import SSESource

# Basic usage with default JSON parser
source = SSESource(
    url="https://api.example.com/events",
    channel_id="sse-events",
)
await kit.attach_source("sse-events", source)
```

The default parser expects SSE `data` fields to contain JSON:

```json
{
    "sender_id": "user123",
    "text": "Hello world",
    "external_id": "msg-456",
    "metadata": {"key": "value"}
}
```

Supported event types: `message`, `msg`, `chat`, or empty (default). Other event types (e.g., `ping`, `heartbeat`) are skipped.

#### Custom SSE Parser

For custom SSE formats, provide a parser function that receives the event type, data, and optional event ID:

```python
from roomkit import InboundMessage, TextContent

def my_parser(event: str, data: str, event_id: str | None) -> InboundMessage | None:
    """Parse custom SSE events."""
    if event != "chat":
        return None  # Only process 'chat' events

    # Parse custom format: "user:message"
    parts = data.split(":", 1)
    if len(parts) < 2:
        return None

    return InboundMessage(
        channel_id="sse-chat",
        sender_id=parts[0],
        content=TextContent(body=parts[1]),
        external_id=event_id,
    )

source = SSESource(
    url="https://stream.example.com/chat",
    channel_id="sse-chat",
    parser=my_parser,
)
```

#### SSESource Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` | `str` | required | SSE endpoint URL |
| `channel_id` | `str` | required | Channel ID for emitted messages |
| `parser` | `Callable` | JSON parser | Function to parse SSE events: `(event, data, id) -> InboundMessage` |
| `headers` | `dict[str, str]` | `None` | HTTP headers (e.g., Authorization) |
| `params` | `dict[str, str]` | `None` | Query parameters for the URL |
| `timeout` | `float` | `30.0` | Connection timeout in seconds |
| `last_event_id` | `str` | `None` | Resume from this event ID (sent as `Last-Event-ID` header) |

#### Resuming from Last Event ID

SSE supports resumption via the `Last-Event-ID` header. SSESource tracks the last received event ID automatically:

```python
# Initial connection
source = SSESource(
    url="https://api.example.com/events",
    channel_id="sse-events",
)
await kit.attach_source("sse-events", source)

# ... connection drops ...

# Get the last event ID for resumption
last_id = source.last_event_id
print(f"Last received event: {last_id}")

# Create new source that resumes from last position
resumed_source = SSESource(
    url="https://api.example.com/events",
    channel_id="sse-events",
    last_event_id=last_id,
)
await kit.attach_source("sse-events", resumed_source)
```

When `auto_restart=True` (default), RoomKit automatically handles reconnection and uses the tracked `last_event_id` for seamless resumption.

#### Authentication

Pass authentication via headers:

```python
source = SSESource(
    url="https://api.example.com/events",
    channel_id="sse-events",
    headers={
        "Authorization": "Bearer your-token-here",
        "X-API-Key": "your-api-key",
    },
)
```

#### Installation

SSESource requires `httpx` and `httpx-sse`:

```bash
pip install roomkit[sse]
```

### WhatsAppPersonalSourceProvider

> **Warning:** This source uses the unofficial WhatsApp Web multidevice protocol
> via [neonize](https://github.com/krypton-byte/neonize).  It is intended for
> **personal use and experimentation only**.  Using unofficial clients may
> violate WhatsApp Terms of Service and could result in account restrictions.

Connect a personal WhatsApp account and receive messages in real time:

```python
from roomkit import RoomKit
from roomkit.sources import WhatsAppPersonalSourceProvider
from roomkit.providers.whatsapp.personal import WhatsAppPersonalProvider

kit = RoomKit()

async def handle_events(event_type: str, data: dict):
    if event_type == "qr":
        print(f"Scan this QR code: {data['codes'][0]}")
    elif event_type == "authenticated":
        print(f"Logged in as {data['jid']}")
    elif event_type == "connected":
        print("WhatsApp connected!")

source = WhatsAppPersonalSourceProvider(
    db="wa-session.db",
    channel_id="wa-personal",
    on_event=handle_events,
)

await kit.attach_source("wa-personal", source)
```

The built-in parser handles text, image, audio (voice notes), video, document,
location, and sticker messages automatically.

#### QR Code Handling

On first connection, WhatsApp requires linking via QR code.  Use the
`on_event` callback to receive QR codes:

```python
async def handle_events(event_type: str, data: dict):
    if event_type == "qr":
        # data["codes"] is a list of QR code strings
        # Display via terminal, web UI, etc.
        print(f"QR: {data['codes'][0]}")
```

After scanning, the session is persisted in the database (SQLite by default).
Subsequent connections reuse the saved session automatically.

#### Custom Parser

Replace the default parser with your own:

```python
from roomkit import InboundMessage, TextContent

async def my_parser(client, event) -> InboundMessage | None:
    info = event.Info
    if info.IsFromMe:
        return None
    return InboundMessage(
        channel_id="wa-personal",
        sender_id=str(info.Sender).split("@")[0],
        content=TextContent(body=event.Message.conversation or ""),
    )

source = WhatsAppPersonalSourceProvider(
    channel_id="wa-personal",
    parser=my_parser,
)
```

#### Bidirectional Pattern (Source + Provider)

Pair the source with `WhatsAppPersonalProvider` for outbound delivery:

```python
from roomkit import RoomKit, WhatsAppPersonalChannel
from roomkit.sources import WhatsAppPersonalSourceProvider
from roomkit.providers.whatsapp.personal import WhatsAppPersonalProvider

kit = RoomKit()

source = WhatsAppPersonalSourceProvider(
    db="wa-session.db",
    channel_id="wa-personal",
    on_event=handle_events,
)

provider = WhatsAppPersonalProvider(source)
kit.register_channel(WhatsAppPersonalChannel("wa-personal", provider=provider))
await kit.attach_source("wa-personal", source, auto_restart=True)
```

#### Session Persistence

By default, neonize stores session state in a local SQLite file.  You can also
use a PostgreSQL URI:

```python
# SQLite (default)
source = WhatsAppPersonalSourceProvider(db="wa-session.db")

# PostgreSQL
source = WhatsAppPersonalSourceProvider(db="postgres://user:pass@host/db")
```

#### Event Callback Reference

| Event | Data | Description |
|-------|------|-------------|
| `qr` | `codes: list[str]` | QR code strings for linking |
| `authenticated` | `jid, user, device` | Successfully paired with WhatsApp |
| `connected` | `{}` | Client connected and ready |
| `disconnected` | `{}` | Connection lost (will reconnect) |
| `logged_out` | `{}` | Session invalidated, re-pairing needed |
| `receipt` | `type, raw_type, chat, sender, sender_name, message_ids, timestamp` | Delivery/read receipt. `type` is human-readable (`delivered`, `read`, `played`, etc.) |
| `presence` | `chat, sender, sender_name, state, media` | Typing indicator. `state`: `composing` or `paused`. `media`: `text` or `audio` |

#### Typing Indicators

Inbound typing indicators are received through the `on_event` callback as
`"presence"` events (requires the client to be "available", which is set
automatically on connect).  Outbound typing can be sent via the source:

```python
# Send composing indicator
await source.send_composing("14155551234@s.whatsapp.net")

# Send recording audio indicator
await source.send_composing("14155551234@s.whatsapp.net", media="audio")

# Stop typing
await source.send_paused("14155551234@s.whatsapp.net")
```

#### Read Receipts

Mark messages as read (send blue ticks):

```python
await source.mark_read(
    message_ids=["ABCD1234"],
    chat="14155551234@s.whatsapp.net",
    sender="14155551234@s.whatsapp.net",
)
```

Inbound receipts are delivered through `on_event` as `"receipt"` events with
`type` values: `delivered`, `read`, `played`, `read_self`, `played_self`, etc.

#### WhatsAppPersonalSourceProvider Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `db` | `str` | `"whatsapp-session.db"` | Database path (SQLite file or PostgreSQL URI) |
| `channel_id` | `str` | `"whatsapp-personal"` | Channel ID for emitted messages |
| `parser` | `Callable` | built-in parser | `(client, event) -> InboundMessage \| None` |
| `on_event` | `Callable` | `None` | `(event_type, data) -> None` lifecycle callback |
| `device_name` | `str` | `"RoomKit"` | Device name in WhatsApp linked devices |
| `device_platform` | `str` | `"chrome"` | Browser prefix in linked devices list (`chrome`, `firefox`, `safari`, `edge`, `desktop`) |
| `self_chat` | `bool` | `False` | Process own messages (for testing/self-chat agents) |

#### Installation

WhatsAppPersonalSourceProvider requires the `neonize` package:

```bash
pip install roomkit[whatsapp-personal]
```

---

## Implementing a Custom Source

Extend `SourceProvider` or `BaseSourceProvider` to create custom sources:

### Minimal Implementation

```python
from roomkit import SourceProvider, SourceStatus, SourceHealth, InboundMessage, TextContent

class MySource(SourceProvider):
    def __init__(self, config: str):
        self._config = config
        self._status = SourceStatus.STOPPED

    @property
    def name(self) -> str:
        return f"my-source:{self._config}"

    @property
    def status(self) -> SourceStatus:
        return self._status

    async def start(self, emit) -> None:
        self._status = SourceStatus.CONNECTING
        # ... connect to external system ...
        self._status = SourceStatus.CONNECTED

        while True:  # Main loop
            # Receive message from external system
            data = await self._receive()

            # Convert to InboundMessage
            message = InboundMessage(
                channel_id="my-channel",
                sender_id=data["from"],
                content=TextContent(body=data["text"]),
                external_id=data["id"],
            )

            # Emit into RoomKit pipeline
            result = await emit(message)
            if result.blocked:
                print(f"Message blocked: {result.reason}")

    async def stop(self) -> None:
        self._status = SourceStatus.STOPPED
        # ... cleanup connections ...
```

### Using BaseSourceProvider

For convenience, extend `BaseSourceProvider` which provides built-in status tracking:

```python
from roomkit import BaseSourceProvider, InboundMessage, TextContent
import asyncio

class WebSocketSource(BaseSourceProvider):
    def __init__(self, url: str):
        super().__init__()
        self._url = url
        self._ws = None

    @property
    def name(self) -> str:
        return f"websocket:{self._url}"

    async def start(self, emit) -> None:
        import websockets

        self._reset_stop()  # Clear stop signal for restart
        self._set_status(SourceStatus.CONNECTING)

        async with websockets.connect(self._url) as ws:
            self._ws = ws
            self._set_status(SourceStatus.CONNECTED)

            while not self._should_stop():
                try:
                    raw = await asyncio.wait_for(ws.recv(), timeout=1.0)
                    data = json.loads(raw)

                    message = InboundMessage(
                        channel_id="websocket",
                        sender_id=data["user_id"],
                        content=TextContent(body=data["text"]),
                    )

                    await emit(message)
                    self._record_message()  # Update stats

                except asyncio.TimeoutError:
                    continue  # Check stop signal

    async def stop(self) -> None:
        await super().stop()  # Sets stop event
        if self._ws:
            await self._ws.close()
```

### BaseSourceProvider Helpers

| Method | Description |
|--------|-------------|
| `_set_status(status, error=None)` | Update status and optionally set error |
| `_record_message()` | Increment message counter and update timestamp |
| `_should_stop()` | Check if `stop()` was called |
| `_reset_stop()` | Clear stop signal (for restarts) |

## Complete Example: NATS Source

```python
from roomkit import (
    RoomKit,
    BaseSourceProvider,
    SourceStatus,
    InboundMessage,
    TextContent,
)
import json

class NATSSource(BaseSourceProvider):
    """Subscribe to NATS subjects for inbound messages."""

    def __init__(self, servers: list[str], subject: str, channel_id: str):
        super().__init__()
        self._servers = servers
        self._subject = subject
        self._channel_id = channel_id
        self._nc = None
        self._sub = None

    @property
    def name(self) -> str:
        return f"nats:{self._subject}"

    async def start(self, emit) -> None:
        import nats

        self._reset_stop()
        self._set_status(SourceStatus.CONNECTING)

        self._nc = await nats.connect(servers=self._servers)
        self._set_status(SourceStatus.CONNECTED)

        async def handler(msg):
            data = json.loads(msg.data.decode())
            inbound = InboundMessage(
                channel_id=self._channel_id,
                sender_id=data.get("sender", "unknown"),
                content=TextContent(body=data.get("text", "")),
                external_id=data.get("id"),
                metadata=data.get("metadata", {}),
            )
            await emit(inbound)
            self._record_message()

        self._sub = await self._nc.subscribe(self._subject, cb=handler)

        # Keep alive until stopped
        while not self._should_stop():
            await asyncio.sleep(1)

    async def stop(self) -> None:
        await super().stop()
        if self._sub:
            await self._sub.unsubscribe()
        if self._nc:
            await self._nc.close()


# Usage
async def main():
    kit = RoomKit()

    source = NATSSource(
        servers=["nats://localhost:4222"],
        subject="chat.inbound.>",
        channel_id="nats-chat",
    )

    await kit.attach_source("nats-chat", source)

    # Process messages via hooks
    @kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC)
    async def log_message(event, context):
        print(f"Received: {event.content}")

    # Run until interrupted
    try:
        while True:
            await asyncio.sleep(1)
    finally:
        await kit.close()
```

## Lifecycle and Cleanup

Sources are automatically cleaned up when calling `kit.close()`:

```python
async with RoomKit() as kit:
    await kit.attach_source("ws", websocket_source)
    await kit.attach_source("nats", nats_source)

    # ... process messages ...

# Both sources automatically stopped and detached
```

## Error Handling

When a source fails:

1. If `auto_restart=True` (default), RoomKit waits `restart_delay` seconds and restarts
2. A `source_error` framework event is emitted
3. Health status changes to `ERROR` or `RECONNECTING`

```python
@kit.on("source_error")
async def handle_source_error(event):
    logger.error(
        "Source %s failed: %s",
        event.data["source_name"],
        event.data["error"],
    )
    # Optionally: alert, metrics, etc.
```

To disable auto-restart for one-shot sources:

```python
await kit.attach_source("one-shot", source, auto_restart=False)
```

---

## Bidirectional Channel Pattern

By design, **SourceProvider handles inbound messages** and **Provider handles outbound messages**. For true bidirectional communication (e.g., a WebSocket that both receives AND sends through RoomKit's pipeline), pair a Source with a Provider that share the same connection.

### Use Case: Multi-Client Chat

Consider a chat application where:
- Browser UI connects via HTTP/WebSocket to your server
- CLI client connects via a separate WebSocket
- Messages from either client should appear in both

```
Browser UI ──HTTP──► Your Server ◄──WebSocket──► CLI Client
                         │
                      RoomKit
                         │
              ┌──────────┴──────────┐
              │                     │
        WebSocketSource      WebSocketProvider
        (CLI → RoomKit)      (RoomKit → CLI)
              │                     │
              └─────── shared ──────┘
                     connection
```

### Implementation

**Step 1: Create a Provider that wraps the Source's send()**

```python
import json
from roomkit import Provider, ProviderCapability, DeliveryResult, OutboundMessage
from roomkit.sources import WebSocketSource, SourceStatus

class WebSocketProvider(Provider):
    """Provider that sends outbound messages through a WebSocketSource."""

    def __init__(self, source: WebSocketSource):
        self._source = source

    @property
    def name(self) -> str:
        return "websocket"

    @property
    def capabilities(self) -> set[ProviderCapability]:
        return {ProviderCapability.TEXT}

    async def send(self, message: OutboundMessage) -> DeliveryResult:
        if self._source.status != SourceStatus.CONNECTED:
            return DeliveryResult(
                success=False,
                error="WebSocket not connected",
            )

        # Serialize to your WebSocket protocol format
        payload = json.dumps({
            "type": "message",
            "sender_id": message.sender_id,
            "text": message.content.body,
            "room_id": message.room_id,
            "timestamp": message.timestamp.isoformat(),
        })

        await self._source.send(payload)
        return DeliveryResult(success=True, external_id=message.id)
```

**Step 2: Wire up Source and Provider**

```python
from roomkit import RoomKit
from roomkit.sources import WebSocketSource

kit = RoomKit()

# Create the source (handles inbound from CLI)
source = WebSocketSource(
    url="wss://cli-gateway.example.com/events",
    channel_id="cli-channel",
)

# Create provider that wraps the source (handles outbound to CLI)
provider = WebSocketProvider(source)

# Register both
kit.register_provider("cli-channel", provider)
await kit.attach_source("cli-channel", source)
```

**Step 3: Prevent echo loops**

Without protection, a message from CLI would broadcast back to CLI. Add a hook to filter:

```python
from roomkit import HookTrigger, HookResult

@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def prevent_echo(event, context):
    # Don't send back to the channel that originated the message
    if event.channel_id == context.target_channel_id:
        return HookResult(blocked=True, reason="echo prevention")
    return HookResult()
```

Or use room-level channel filtering if you want more control:

```python
# When processing inbound, track the source
@kit.hook(HookTrigger.BEFORE_INBOUND)
async def tag_source(event, context):
    context.metadata["source_channel"] = event.channel_id
    return HookResult()

@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def skip_source_channel(event, context):
    source_channel = context.metadata.get("source_channel")
    if source_channel == context.target_channel_id:
        return HookResult(blocked=True)
    return HookResult()
```

### Complete Example: Browser + CLI Chat

```python
import asyncio
from fastapi import FastAPI, WebSocket
from roomkit import RoomKit, HookTrigger, HookResult, InboundMessage, TextContent
from roomkit.sources import WebSocketSource, SourceStatus

app = FastAPI()
kit = RoomKit()

# --- CLI WebSocket Channel ---
cli_source = WebSocketSource(
    url="wss://cli-gateway.example.com/events",
    channel_id="cli",
)
cli_provider = WebSocketProvider(cli_source)
kit.register_provider("cli", cli_provider)

# --- Echo Prevention ---
@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def prevent_echo(event, context):
    if event.channel_id == context.target_channel_id:
        return HookResult(blocked=True, reason="echo")
    return HookResult()

# --- Browser WebSocket Endpoint ---
@app.websocket("/chat/{room_id}/{user_id}")
async def browser_chat(websocket: WebSocket, room_id: str, user_id: str):
    await websocket.accept()

    # Subscribe to room broadcasts for this browser
    async def send_to_browser(event):
        await websocket.send_json({
            "type": "message",
            "sender_id": event.sender_id,
            "text": event.content.body,
        })

    # Use hooks or realtime subscription to forward messages
    # (simplified - in production use kit.subscribe_room or similar)

    try:
        while True:
            data = await websocket.receive_json()

            # Process browser message through RoomKit
            await kit.process_inbound(InboundMessage(
                channel_id="browser",
                sender_id=user_id,
                content=TextContent(body=data["text"]),
                room_id=room_id,
            ))
    except Exception:
        pass

@app.on_event("startup")
async def startup():
    await kit.attach_source("cli", cli_source)

@app.on_event("shutdown")
async def shutdown():
    await kit.close()
```

### Message Flow

```
CLI sends "Hello":
  CLI ──WebSocket──► WebSocketSource.emit()
                          │
                          ▼
                    kit.process_inbound()
                          │
                          ▼
                    BEFORE_BROADCAST hook (echo check)
                          │
                    ┌─────┴─────┐
                    ▼           ▼
              cli channel   browser channel
              (blocked:     (delivered via
               echo)         HTTP/WS)

Browser sends "Hi":
  Browser ──HTTP──► kit.process_inbound()
                          │
                          ▼
                    BEFORE_BROADCAST hook
                          │
                    ┌─────┴─────┐
                    ▼           ▼
              cli channel   browser channel
              (delivered    (blocked:
               via WS        echo)
               Provider)
```

### Why Source + Provider Pair?

| Alternative | Problem |
|-------------|---------|
| AFTER_BROADCAST hook with `source.send()` | Bypasses delivery tracking, retries, circuit breakers |
| Single "bidirectional source" | Conflates inbound/outbound concerns, harder to test |
| Custom channel delivery | Reinvents what Provider already does |

The **Source + Provider pair** pattern:
- Uses RoomKit's existing abstractions correctly
- Gets delivery tracking, error handling, and observability for free
- Keeps inbound and outbound concerns separated
- Allows different retry/circuit breaker policies per direction

---

## API Reference

::: roomkit.SourceStatus

::: roomkit.SourceHealth

::: roomkit.SourceProvider

::: roomkit.BaseSourceProvider

::: roomkit.EmitCallback

::: roomkit.SourceAlreadyAttachedError

::: roomkit.SourceNotFoundError
