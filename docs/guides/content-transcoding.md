# Content Transcoding

When a room has multiple channel types (SMS, WhatsApp, Email, Voice), content must be adapted to each channel's capabilities. RoomKit handles this automatically during broadcast via the `ContentTranscoder`.

## How It Works

During broadcast, the `EventRouter`:

1. Checks if the target channel natively supports the content type
2. If not, calls `ContentTranscoder.transcode()` to adapt the content
3. Applies `max_length` enforcement after transcoding
4. If transcoding fails (returns `None`), the target channel is skipped

```
RichContent (buttons, cards)
  ├── WhatsApp: passes through (supports RICH)
  ├── SMS: transcoded → TextContent (plain_text fallback)
  └── Email: passes through (supports RICH)
```

## Fallback Chains

The `DefaultContentTranscoder` applies intelligent fallbacks:

| Source Content | Target Supports | Fallback |
|---------------|----------------|----------|
| `TextContent` | Always | N/A (universal) |
| `RichContent` | `RICH` in media_types | `plain_text` field or `body` as TextContent |
| `MediaContent` | `MEDIA` | `[Media: {caption or filename or url}]` |
| `AudioContent` | `AUDIO` | `transcript` field, or `[Voice message: {url}]` |
| `VideoContent` | `VIDEO` | `[Video: {url}]` |
| `LocationContent` | `LOCATION` | `[Location: {label} ({lat}, {lon})]` |
| `CompositeContent` | varies | Recursive transcode; flatten if all parts become text |
| `TemplateContent` | `TEMPLATE` | `body` field or `[Template: {id}]` |
| `EditContent` | `supports_edit` | Transcode `new_content` + "Correction:" prefix |
| `DeleteContent` | `supports_delete` | `[Message deleted]` |
| `SystemContent` | Always | Pass-through |

## ChannelMediaType

Channels declare which content types they support:

```python
from roomkit.models.enums import ChannelMediaType

# Values: TEXT, RICH, MEDIA, AUDIO, VIDEO, LOCATION, TEMPLATE
```

## ChannelCapabilities

Each channel binding declares its capabilities:

```python
from roomkit.models.channel import ChannelCapabilities, ChannelMediaType

sms_caps = ChannelCapabilities(
    media_types=[ChannelMediaType.TEXT],       # Text only
    max_length=160,                             # SMS character limit
    supports_edit=False,
    supports_delete=False,
)

whatsapp_caps = ChannelCapabilities(
    media_types=[ChannelMediaType.TEXT, ChannelMediaType.RICH, ChannelMediaType.MEDIA,
                 ChannelMediaType.LOCATION, ChannelMediaType.TEMPLATE],
    supports_edit=True,
    supports_delete=True,
    supports_rich_text=True,
    supports_buttons=True,
    supports_quick_replies=True,
)
```

## CompositeContent Handling

`CompositeContent` (multiple parts in one message) is recursively transcoded:

```python
from __future__ import annotations

from roomkit.models.events import CompositeContent, LocationContent, MediaContent, TextContent

# A composite with text + image + location
message = CompositeContent(parts=[
    TextContent(body="Here's the restaurant:"),
    MediaContent(url="https://example.com/photo.jpg", caption="Restaurant entrance"),
    LocationContent(latitude=48.8566, longitude=2.3522, label="Le Petit Bistro"),
])

# Sent to SMS (TEXT only):
# → Each part is transcoded individually
# → All parts become text → flattened into single TextContent:
# "Here's the restaurant:\n[Media: Restaurant entrance]\n[Location: Le Petit Bistro (48.8566, 2.3522)]"
```

**Smart flattening**: If all parts become `TextContent` after transcoding, they are merged into a single `TextContent` joined by newlines.

## max_length Enforcement

After transcoding, the `EventRouter` enforces `max_length` from channel capabilities. This is applied **after** transcoding, so a rich message that falls back to text is truncated if needed:

```
RichContent (500 chars body) → SMS (max_length=160) → TextContent truncated to 160 chars
```

## Custom Transcoder

Implement `ContentTranscoder` for custom logic:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.core.router import ContentTranscoder
from roomkit.models.channel import ChannelBinding
from roomkit.models.events import EventContent, RichContent, TextContent


class WhatsAppStyleTranscoder(ContentTranscoder):
    async def transcode(
        self,
        content: EventContent,
        source_binding: ChannelBinding,
        target_binding: ChannelBinding,
    ) -> EventContent | None:
        # Custom: convert RichContent to WhatsApp bold markdown
        if isinstance(content, RichContent):
            return TextContent(body=f"*{content.plain_text or content.body}*")
        # Fall through for other types
        return content


kit = RoomKit(transcoder=WhatsAppStyleTranscoder())
```

Return `None` to signal that the content cannot be represented on the target channel — the channel will be skipped for this message.

## Practical Example: Multichannel Room

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.channels import EmailChannel, SMSChannel, WebSocketChannel
from roomkit.models.events import RichContent

kit = RoomKit()
kit.register_channel(SMSChannel("sms", provider=twilio))
kit.register_channel(EmailChannel("email", provider=sendgrid))
kit.register_channel(WebSocketChannel("ws"))

# All three channels attached to the same room
await kit.attach_channel("room-1", "sms")
await kit.attach_channel("room-1", "email")
await kit.attach_channel("room-1", "ws")

# A rich message sent from WebSocket:
# RichContent(body="**Important update**", format="markdown",
#             plain_text="Important update", buttons=[...])
#
# → WebSocket: receives full RichContent (supports RICH)
# → Email: receives full RichContent (supports RICH)
# → SMS: receives TextContent(body="Important update") — plain_text fallback
```

## Content Types Reference

| Type | Key Fields | Typical Channels |
|------|-----------|-----------------|
| `TextContent` | `body`, `language` | All |
| `RichContent` | `body`, `format`, `plain_text`, `buttons`, `cards`, `quick_replies` | WhatsApp, Messenger, Teams, Email, WebSocket |
| `MediaContent` | `url`, `mime_type`, `filename`, `caption` | WhatsApp, Messenger, Email, Telegram |
| `AudioContent` | `url`, `mime_type`, `duration_seconds`, `transcript` | WhatsApp, Telegram, Voice |
| `VideoContent` | `url`, `mime_type`, `duration_seconds`, `thumbnail_url` | WhatsApp, Telegram |
| `LocationContent` | `latitude`, `longitude`, `label`, `address` | WhatsApp, Telegram, Messenger |
| `CompositeContent` | `parts[]` (max depth=5) | Varies per part |
| `TemplateContent` | `template_id`, `language`, `parameters`, `body` | WhatsApp, RCS |
| `EditContent` | `target_event_id`, `new_content` | WhatsApp, Telegram, WebSocket |
| `DeleteContent` | `target_event_id`, `delete_type`, `reason` | WhatsApp, Telegram, WebSocket |
| `SystemContent` | `body`, `code`, `data` | All (internal) |
