# Event-Driven Sources

RoomKit's source system enables **event-driven message ingestion** from persistent connections like WebSockets, NATS, SSE, or custom protocols (e.g., WhatsApp via neonize).

## Overview

Unlike webhook-based providers that receive HTTP POST requests, **SourceProviders** maintain persistent connections and push messages into RoomKit as they arrive.

```python
from roomkit import RoomKit, InboundMessage
from roomkit.sources.base import SourceProvider, SourceStatus

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
from roomkit.sources.base import SourceStatus

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
from roomkit import InboundMessage, TextContent
from roomkit.sources.base import SourceProvider, SourceStatus, SourceHealth

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
            if result.action == "block":
                print(f"Message blocked: {result.reason}")

    async def stop(self) -> None:
        self._status = SourceStatus.STOPPED
        # ... cleanup connections ...
```

### Using BaseSourceProvider

For convenience, extend `BaseSourceProvider` which provides built-in status tracking:

```python
from roomkit import InboundMessage, TextContent
from roomkit.sources.base import BaseSourceProvider
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
from roomkit import RoomKit, InboundMessage, TextContent
from roomkit.sources.base import BaseSourceProvider, SourceStatus
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

By design, **a SourceProvider brings messages in** and **a provider sends them
out**. When one connection carries both directions (a WebSocket that both
receives and sends), the channel's provider sends through the source.

### Use Case: Multi-Client Chat

A browser user and a CLI client share one room. The browser connects to your
server; the CLI talks to a gateway that RoomKit reaches over a WebSocket:

```
Browser ──WebSocket──► Your server ◄──WebSocket──► CLI gateway
                           │
                        RoomKit
                           │
             ┌─────────────┴─────────────┐
             │                           │
   WebSocketChannel "browser"     TransportChannel "cli"
                                   ├── WebSocketSource       (gateway → RoomKit)
                                   └── WebSocketSendProvider (RoomKit → gateway)
                                         shared connection
```

### Implementation

**Step 1: A provider that sends through the source**

`TransportChannel` calls its provider's `send(event, to)`. Subclassing
`HTTPProvider` gives that shape:

```python
import json

from roomkit import RoomEvent
from roomkit.models.delivery import ProviderResult
from roomkit.providers.http.base import HTTPProvider
from roomkit.sources import SourceStatus, WebSocketSource


class WebSocketSendProvider(HTTPProvider):
    """Sends outbound events through the source's WebSocket connection."""

    def __init__(self, source: WebSocketSource) -> None:
        self._source = source

    async def send(self, event: RoomEvent, to: str) -> ProviderResult:
        if self._source.status != SourceStatus.CONNECTED:
            return ProviderResult(success=False, error="websocket_not_connected")
        await self._source.send(json.dumps({
            "type": "message",
            "room_id": event.room_id,
            "sender_id": event.source.participant_id,
            "text": event.content.body,
        }))
        return ProviderResult(success=True, provider_message_id=event.id)
```

**Step 2: Wire the channel, the source and the room**

```python
from roomkit import ChannelType, RoomEvent, RoomKit, WebSocketChannel
from roomkit.channels import TransportChannel

kit = RoomKit()

# The CLI side: inbound from the source, outbound through the provider
source = WebSocketSource(url="wss://cli-gateway.example.com/events", channel_id="cli")
cli = TransportChannel(
    "cli",
    ChannelType.WEBSOCKET,
    provider=WebSocketSendProvider(source),
    requires_recipient=False,  # the connection is the address
)

# The browser side
browser = WebSocketChannel("browser")

kit.register_channel(cli)
kit.register_channel(browser)

await kit.create_room(room_id="chat")
await kit.attach_channel("chat", "cli")
await kit.attach_channel("chat", "browser")

async def to_browser(connection_id: str, event: RoomEvent) -> None:
    await browser_sockets[connection_id].send_json(event.model_dump(mode="json"))

browser.register_connection("tab-1", to_browser, room_id="chat")
await kit.attach_source("cli", source)
```

A gateway message such as `{"sender_id": "cli-user", "text": "Hello"}` goes
through the inbound pipeline and reaches the browser. A browser message
(`kit.process_inbound(InboundMessage(channel_id="browser", ...))`) reaches the
gateway through `WebSocketSendProvider`.

**No echo filter needed:** the router never delivers an event back to the
channel it came from, so the gateway does not receive its own messages.

### Why Source + Provider Pair?

| Alternative | Problem |
|-------------|---------|
| AFTER_BROADCAST hook with `source.send()` | Bypasses the router: no retry policy, rate limit, circuit breaker or delivery status |
| Single "bidirectional source" | Conflates inbound and outbound concerns, harder to test |
| A custom `Channel` subclass | Reimplements the delivery `TransportChannel` already does |

The **Source + Provider pair**:

- Keeps inbound and outbound separate, each testable alone
- Delivers through the router, with its retries, rate limits and circuit breaker
- Reports delivery results the same way as SMS or email channels

---

## API Reference

::: roomkit.sources.base.SourceStatus

::: roomkit.sources.base.SourceHealth

::: roomkit.sources.base.SourceProvider

::: roomkit.sources.base.BaseSourceProvider

::: roomkit.sources.base.EmitCallback

::: roomkit.SourceAlreadyAttachedError

::: roomkit.SourceNotFoundError
