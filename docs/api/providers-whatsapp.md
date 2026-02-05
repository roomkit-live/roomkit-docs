# WhatsApp Providers

## Business API (Webhook-based)

::: roomkit.WhatsAppProvider

::: roomkit.MockWhatsAppProvider

## Personal Account (Neonize)

> **Warning:** `WhatsAppPersonalProvider` uses the unofficial WhatsApp Web
> multidevice protocol via [neonize](https://github.com/krypton-byte/neonize).
> It is intended for **personal use and experimentation only**.  Using
> unofficial clients may violate WhatsApp Terms of Service and could result in
> account restrictions.

`WhatsAppPersonalProvider` sends outbound messages through a shared neonize
client managed by `WhatsAppPersonalSourceProvider`.  The provider does **not**
import neonize at the module level, so importing the class never requires the
optional dependency.

### Quick start

```python
from roomkit import RoomKit, WhatsAppPersonalChannel
from roomkit.sources import WhatsAppPersonalSourceProvider
from roomkit.providers.whatsapp.personal import WhatsAppPersonalProvider

kit = RoomKit()

# Lifecycle events (QR code, auth, connection status)
async def on_wa_event(event_type: str, data: dict):
    if event_type == "qr":
        print(f"Scan QR: {data['codes'][0]}")
    elif event_type == "connected":
        print("WhatsApp connected!")

# Source owns the neonize client and inbound message loop
source = WhatsAppPersonalSourceProvider(
    db="wa-session.db",
    channel_id="wa-personal",
    on_event=on_wa_event,
)

# Provider wraps the source for outbound delivery
provider = WhatsAppPersonalProvider(source)

# Register channel and attach source
kit.register_channel(WhatsAppPersonalChannel("wa-personal", provider=provider))
await kit.attach_source("wa-personal", source, auto_restart=True)
```

### Supported outbound content types

| Content type | Neonize method | Notes |
|-------------|----------------|-------|
| `TextContent` | `send_message` | Plain text |
| `MediaContent` (image/*) | `send_image` | With optional caption |
| `MediaContent` (other) | `send_document` | With filename |
| `AudioContent` | `send_audio` | `ptt=True` for voice notes (audio/ogg) |
| `VideoContent` | `send_video` | MP4 |
| `LocationContent` | `send_location` | Lat/lng with optional label |

Unsupported content types return `ProviderResult(success=False)`.

### Typing indicators

Send typing indicators to show "composing..." or "recording audio..." on the
recipient's device:

```python
provider = WhatsAppPersonalProvider(source)

# Show "typing..." to recipient
await provider.send_typing("14155551234", is_typing=True)

# Show "recording audio..." to recipient
await provider.send_typing("14155551234", is_typing=True, media="audio")

# Stop typing indicator
await provider.send_typing("14155551234", is_typing=False)
```

Inbound typing indicators are delivered through the `on_event` callback as
`"presence"` events:

```python
async def on_wa_event(event_type: str, data: dict):
    if event_type == "presence":
        name = data.get("sender_name") or data["sender"]
        if data["state"] == "composing":
            action = "recording audio..." if data["media"] == "audio" else "typing..."
            print(f"[{name}] {action}")
        elif data["state"] == "paused":
            print(f"[{name}] stopped typing")
```

> **Note:** Inbound typing requires the client to be marked as "available".
> This is done automatically on connect.  Typing indicators are not sent in
> self-chat — a second person must be typing in your conversation.

### Read receipts

Send read receipts (blue ticks) and receive delivery/read notifications:

```python
# Send blue ticks for a message
await provider.mark_read(
    message_ids=["ABCD1234"],
    chat="14155551234",
    sender="14155551234",
)
```

Inbound receipts are delivered through the `on_event` callback:

```python
async def on_wa_event(event_type: str, data: dict):
    if event_type == "receipt":
        # data["type"]: "delivered", "read", "played", "read_self", etc.
        # data["message_ids"]: list of message IDs
        # data["sender_name"]: resolved name (if known)
        print(f"Receipt: {data['type']} from {data.get('sender_name') or data['sender']}")
```

| Receipt type | Meaning |
|-------------|---------|
| `delivered` | Message reached WhatsApp servers (grey double ticks) |
| `read` | Message was read by recipient (blue double ticks) |
| `read_self` | Message was read on another linked device |
| `played` | Audio/video was played by recipient |
| `played_self` | Audio/video played on another linked device |
| `sender` | Sender acknowledgement |
| `server_error` | Server delivery error |

### Installation

```bash
pip install roomkit[whatsapp-personal]
```

### API reference

::: roomkit.WhatsAppPersonalProvider
