# SMS Providers

## Webhook Integration (Recommended)

The simplest way to handle SMS webhooks from any provider is using `extract_sms_meta()` combined with `kit.process_webhook()`:

```python
from roomkit import RoomKit, extract_sms_meta, parse_voicemeup_webhook

kit = RoomKit()

@app.post("/webhooks/sms/{provider}/inbound")
async def sms_webhook(provider: str, payload: dict):
    channel_id = f"sms-{provider}"

    # VoiceMeUp needs special handling for MMS aggregation
    if provider == "voicemeup":
        message = parse_voicemeup_webhook(payload, channel_id=channel_id)
        if message is None:
            return {"ok": True, "buffered": True}  # Waiting for image part
        await kit.process_inbound(message)
        return {"ok": True}

    # All other providers: generic path
    meta = extract_sms_meta(provider, payload)

    if meta.is_inbound:
        await kit.process_inbound(meta.to_inbound(channel_id))
    elif meta.is_status:
        await kit.process_delivery_status(meta.to_status())

    return {"ok": True}
```

### Delivery Status Tracking

Providers send webhooks for delivery status updates (sent, delivered, failed). Register a handler to track these:

```python
from roomkit import DeliveryStatus

@kit.on_delivery_status
async def track_delivery(status: DeliveryStatus):
    if status.status == "delivered":
        logger.info("Message %s delivered to %s", status.message_id, status.recipient)
    elif status.status == "failed":
        logger.error("Message %s failed: %s", status.message_id, status.error_message)
```

## Base Classes

::: roomkit.SMSProvider

::: roomkit.MockSMSProvider

## MMS Handling

RoomKit supports MMS (Multimedia Messaging Service) through SMS providers. When an MMS arrives, the channel type is automatically set to `mms` instead of `sms`.

### Media URL Re-hosting

**Important**: Provider media URLs (from VoiceMeUp, Twilio, etc.) should be re-hosted before forwarding to other channels. This is necessary because:

- Provider URLs may be temporary and expire
- Some providers block certain services (e.g., AI providers cannot fetch VoiceMeUp URLs)
- You maintain control over media availability

**Recommended**: Use a `BEFORE_BROADCAST` hook to automatically re-host media URLs before they reach AI channels. See [VoiceMeUp Media URL Restrictions](#voicemeup-media-url-restrictions) for a complete example.

### Provider Comparison

| Provider | MMS Support | Webhook Format | Notes |
|----------|-------------|----------------|-------|
| Twilio | Single webhook | `MediaUrl0`, `MediaUrl1`... | Up to 10 media per message |
| Sinch | Single webhook | `media[]` array | Full MMS support |
| Telnyx | Single webhook | `media_urls[]` array | Full MMS support |
| VoiceMeUp | **Split webhooks** | See below | Requires aggregation |

## Sinch

::: roomkit.SinchSMSProvider

::: roomkit.SinchConfig

::: roomkit.parse_sinch_webhook

## Telnyx

::: roomkit.TelnyxSMSProvider

::: roomkit.TelnyxConfig

## Twilio

::: roomkit.TwilioSMSProvider

::: roomkit.TwilioConfig

::: roomkit.parse_twilio_webhook

## VoiceMeUp

::: roomkit.VoiceMeUpSMSProvider

::: roomkit.VoiceMeUpConfig

::: roomkit.parse_voicemeup_webhook

### VoiceMeUp MMS Split Webhooks

VoiceMeUp handles MMS differently from other providers. When a user sends an MMS (text + image), VoiceMeUp sends **two separate webhooks**:

1. **First webhook**: Contains the text message + a `.mms.html` metadata wrapper (not the actual image)
2. **Second webhook**: Contains the actual image (no text)

Both webhooks have **different `sms_hash` values**, making correlation non-trivial.

**Without aggregation**, you get two separate events — confusing for AI channels that receive the text question in one event and the image in another.

### Automatic MMS Aggregation

`parse_voicemeup_webhook()` automatically handles MMS aggregation. When the first webhook arrives (text + `.mms.html`), it buffers the message and returns `None`. When the second webhook arrives (image), it merges them and returns a complete `InboundMessage`.

**Usage example**:

```python
from roomkit import parse_voicemeup_webhook, configure_voicemeup_mms

# Optional: configure timeout behavior (default: 5s)
configure_voicemeup_mms(
    timeout_seconds=5.0,
    on_timeout=handle_orphaned_text,  # Called if image never arrives
)

@app.post("/webhooks/sms/voicemeup/inbound")
async def voicemeup_webhook(payload: dict):
    message = parse_voicemeup_webhook(payload, channel_id="sms")

    if message is None:
        # First part buffered, waiting for image
        return {"ok": True, "buffered": True}

    # Complete message (SMS or merged MMS)
    await kit.process_inbound(message)
    return {"ok": True}
```

::: roomkit.configure_voicemeup_mms

**How it works**:

| Webhook | Content | Action |
|---------|---------|--------|
| First (`.mms.html`) | Text + metadata | Buffer, return `None` |
| Second (image) | Media only | Merge with buffered text, return `InboundMessage` |
| Timeout | - | Emit text-only via `on_timeout` callback |

**Correlation key**: `source_number:destination_number:datetime_transmission`

### VoiceMeUp Media URL Restrictions

VoiceMeUp's CDN (`clients.voicemeup.com`) blocks certain services via `robots.txt`, including Anthropic's image fetcher. **Always re-host VoiceMeUp media** before sending to AI channels.

**Recommended: Use a filtered BEFORE_BROADCAST hook** to automatically re-host media URLs only for inbound MMS:

```python
import httpx
from urllib.parse import urlparse
from roomkit import (
    RoomKit, HookTrigger, HookResult, RoomEvent, RoomContext,
    ChannelType, ChannelDirection, MediaContent, CompositeContent,
)

# Domains that need re-hosting
REHOST_DOMAINS = {"clients.voicemeup.com", "dev-clients.voicemeup.com"}

def needs_rehosting(url: str) -> bool:
    return urlparse(url).netloc in REHOST_DOMAINS

async def rehost_url(url: str) -> str:
    """Download media and upload to your storage."""
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.get(url)
        response.raise_for_status()
        # Upload to S3, Cloudflare R2, or local storage
        new_url = await upload_to_storage(response.content)
        return new_url

async def rehost_media_hook(event: RoomEvent, ctx: RoomContext) -> HookResult:
    """Re-host provider media URLs before broadcast to AI channels."""
    content = event.content

    if isinstance(content, MediaContent) and needs_rehosting(content.url):
        event.content = MediaContent(
            url=await rehost_url(content.url),
            mime_type=content.mime_type,
            caption=content.caption,
        )
    elif isinstance(content, CompositeContent):
        new_parts = []
        for part in content.parts:
            if isinstance(part, MediaContent) and needs_rehosting(part.url):
                new_parts.append(MediaContent(
                    url=await rehost_url(part.url),
                    mime_type=part.mime_type,
                    caption=part.caption,
                ))
            else:
                new_parts.append(part)
        event.content = CompositeContent(parts=new_parts)

    return HookResult.allow()

# Register the hook with filters — only runs for inbound SMS/MMS
kit = RoomKit()
kit.hook(
    HookTrigger.BEFORE_BROADCAST,
    name="rehost_media",
    channel_types={ChannelType.SMS, ChannelType.MMS},
    directions={ChannelDirection.INBOUND},
)(rehost_media_hook)
```

**Hook filtering** ensures the hook only runs for relevant events:
- `channel_types` — Only SMS and MMS events (not email, websocket, etc.)
- `directions` — Only inbound events (not AI responses going outbound)
- `channel_ids` — Optionally restrict to specific channel IDs (e.g., `{"sms-voicemeup"}`)

## Utilities

### Webhook Metadata

::: roomkit.WebhookMeta

::: roomkit.extract_sms_meta

### Delivery Status

::: roomkit.DeliveryStatus

### Phone Normalization

::: roomkit.normalize_phone

::: roomkit.is_valid_phone
