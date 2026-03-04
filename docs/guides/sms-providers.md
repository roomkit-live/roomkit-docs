# SMS & RCS Providers

RoomKit supports 4 SMS providers (Twilio, Telnyx, Sinch, VoiceMeUp) and 2 RCS providers (Twilio RCS, Telnyx RCS). All providers implement the `SMSProvider` or `RCSProvider` ABC and use async HTTP communication.

## Quick Start

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.channels import SMSChannel
from roomkit.providers.twilio import TwilioConfig, TwilioSMSProvider

provider = TwilioSMSProvider(
    TwilioConfig(
        account_sid="ACxxxxxxxxxx",
        auth_token="your-auth-token",
        from_number="+15551234567",
    )
)

sms = SMSChannel("sms-main", provider=provider)

kit = RoomKit()
kit.register_channel(sms)

await kit.create_room(room_id="r1")
await kit.attach_channel("r1", "sms-main", metadata={"phone_number": "+15559999999"})
```

---

## Twilio

The most widely used SMS provider. Supports SMS, MMS, and Messaging Services.

```python
from __future__ import annotations

from roomkit.providers.twilio import TwilioConfig, TwilioSMSProvider

provider = TwilioSMSProvider(
    TwilioConfig(
        account_sid="ACxxxxxxxxxx",
        auth_token="your-auth-token",
        from_number="+15551234567",
        messaging_service_sid="MGxxxxxxxxxx",    # Optional, recommended for production
        timeout=10.0,
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `account_sid` | required | Twilio account SID |
| `auth_token` | required | Auth token (stored as `SecretStr`) |
| `from_number` | required | Default sender number (E.164) |
| `messaging_service_sid` | `None` | Messaging Service ID (better rate limits, A/B testing) |
| `timeout` | `10.0` | HTTP timeout in seconds |

### Webhook Handling

Twilio sends **form-encoded** webhooks (unlike other providers which use JSON):

```python
from __future__ import annotations

from roomkit.providers.twilio.sms import parse_twilio_webhook

# In your web framework (e.g., FastAPI)
@app.post("/webhook/twilio")
async def twilio_webhook(request: Request):
    payload = dict(await request.form())

    # Verify signature (recommended)
    is_valid = provider.verify_signature(
        payload=await request.body(),
        signature=request.headers.get("X-Twilio-Signature"),
        url=str(request.url),
    )
    if not is_valid:
        return Response(status_code=403)

    inbound = parse_twilio_webhook(payload, channel_id="sms-main")
    if inbound:
        await kit.process_inbound(inbound)

    return Response(content="<Response/>", media_type="application/xml")
```

**Signature verification**: HMAC-SHA1 based. Requires the full webhook URL.

**MMS support**: Up to 10 media URLs per message, automatically detected from webhook fields `NumMedia`, `MediaUrl0`, etc.

---

## Telnyx

Modern API with ED25519 signature verification and JSON webhooks.

```python
from __future__ import annotations

from roomkit.providers.telnyx import TelnyxConfig, TelnyxSMSProvider

provider = TelnyxSMSProvider(
    TelnyxConfig(
        api_key="KEYxxxxxxxxxx",
        from_number="+15551234567",
        messaging_profile_id="xxxxxxxxxx",    # Optional
        timeout=10.0,
    ),
    public_key="your-public-key",              # For webhook verification
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `api_key` | required | Telnyx API key (Bearer token) |
| `from_number` | required | Default sender (E.164) |
| `messaging_profile_id` | `None` | Messaging profile for webhook routing |
| `public_key` | `None` | ED25519 public key for webhook verification |

### Webhook Handling

```python
from __future__ import annotations

from roomkit.providers.telnyx.sms import parse_telnyx_webhook

@app.post("/webhook/telnyx")
async def telnyx_webhook(request: Request):
    payload = await request.json()

    # Verify ED25519 signature
    is_valid = provider.verify_signature(
        payload=await request.body(),
        signature=request.headers.get("Telnyx-Signature-ed25519"),
        timestamp=request.headers.get("Telnyx-Timestamp"),
    )

    inbound = parse_telnyx_webhook(payload, channel_id="sms-main", strict=True)
    if inbound:
        await kit.process_inbound(inbound)

    return {"status": "ok"}
```

**Webhook events**: `message.received`, `message.sent`, `message.delivered`, `message.failed`

**Signature**: ED25519 — more secure than HMAC. Requires `PyNaCl` library.

---

## Sinch

Regional API endpoints for compliance and low latency.

```python
from __future__ import annotations

from roomkit.providers.sinch import SinchConfig, SinchSMSProvider

provider = SinchSMSProvider(
    SinchConfig(
        service_plan_id="your-plan-id",
        api_token="your-api-token",
        from_number="+15551234567",
        region="us",                          # us, eu, au, br, ca
        webhook_secret="your-webhook-secret",
        timeout=10.0,
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `service_plan_id` | required | Sinch service plan ID |
| `api_token` | required | API token (Bearer) |
| `from_number` | required | Default sender (E.164) |
| `region` | `"us"` | API region: `us`, `eu`, `au`, `br`, `ca` |
| `webhook_secret` | `None` | HMAC-SHA1 webhook verification secret |

### Webhook Handling

```python
from __future__ import annotations

from roomkit.providers.sinch.sms import parse_sinch_webhook

@app.post("/webhook/sinch")
async def sinch_webhook(request: Request):
    payload = await request.json()

    is_valid = provider.verify_signature(
        payload=await request.body(),
        signature=request.headers.get("X-Sinch-Signature"),
    )

    inbound = parse_sinch_webhook(payload, channel_id="sms-main")
    if inbound:
        await kit.process_inbound(inbound)

    return {"status": "ok"}
```

**Regional endpoints**: API URL is `https://{region}.sms.api.sinch.com/xms/v1/{plan_id}/batches`. Choose the region closest to your users for lower latency.

**MMS**: Supports media via `type: mt_media` with flexible single/multiple media handling.

---

## VoiceMeUp

Canadian provider with unique split-webhook MMS handling and long message auto-segmentation.

```python
from __future__ import annotations

from roomkit.providers.voicemeup import VoiceMeUpConfig, VoiceMeUpSMSProvider

provider = VoiceMeUpSMSProvider(
    VoiceMeUpConfig(
        username="your-username",
        auth_token="your-auth-token",
        from_number="+15145551234",
        environment="production",             # or "sandbox"
        timeout=10.0,
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `username` | required | API username |
| `auth_token` | required | Auth token |
| `from_number` | required | Default sender (E.164) |
| `environment` | `"production"` | `"production"` or `"sandbox"` for testing |

### MMS Aggregation

VoiceMeUp sends MMS as **two separate webhooks** — a text/metadata wrapper first, then the media attachment. RoomKit handles this automatically:

```python
from __future__ import annotations

from roomkit.providers.voicemeup.sms import VoiceMeUpSMSProvider

# Configure MMS aggregation
async def on_mms_timeout(msg):
    logger.warning(f"MMS image never arrived for {msg.sender_id}")

provider.configure_mms(timeout_seconds=5.0, on_timeout=on_mms_timeout)


@app.post("/webhook/voicemeup")
async def voicemeup_webhook(request: Request):
    payload = await request.json()

    # Returns None if buffering (waiting for second part)
    # Returns InboundMessage when complete
    inbound = provider.parse_mms_webhook(payload, channel_id="sms-main")
    if inbound:
        await kit.process_inbound(inbound)

    return {"status": "ok"}
```

**Long messages**: Auto-segmented at 1000 characters.

**Media**: One attachment per message.

**Sandbox**: Use `environment="sandbox"` for development — routes to `dev-clients.voicemeup.com`.

---

## RCS Providers

RCS (Rich Communication Services) extends SMS with rich content support — buttons, cards, read receipts, and typing indicators.

### Twilio RCS

```python
from __future__ import annotations

from roomkit.channels import RCSChannel
from roomkit.providers.twilio.rcs import TwilioRCSConfig, TwilioRCSProvider

provider = TwilioRCSProvider(
    TwilioRCSConfig(
        account_sid="ACxxxxxxxxxx",
        auth_token="your-auth-token",
        messaging_service_sid="MGxxxxxxxxxx",    # Must be RCS-enabled
        timeout=10.0,
    )
)

rcs = RCSChannel("rcs-main", provider=provider, fallback=True)
kit.register_channel(rcs)
```

**SMS fallback**: When `fallback=True` (default), recipients without RCS automatically receive SMS instead.

```python
from __future__ import annotations

from roomkit.providers.twilio.rcs import parse_twilio_rcs_webhook

@app.post("/webhook/twilio-rcs")
async def rcs_webhook(request: Request):
    payload = dict(await request.form())
    inbound = parse_twilio_rcs_webhook(payload, channel_id="rcs-main")
    if inbound:
        await kit.process_inbound(inbound)
    return Response(content="<Response/>", media_type="application/xml")
```

### Telnyx RCS

Includes RCS capability checking — verify if a phone number supports RCS before sending.

```python
from __future__ import annotations

from roomkit.channels import RCSChannel
from roomkit.providers.telnyx.rcs import TelnyxRCSConfig, TelnyxRCSProvider

provider = TelnyxRCSProvider(
    TelnyxRCSConfig(
        api_key="KEYxxxxxxxxxx",
        agent_id="your-rcs-agent-id",          # After brand verification
        messaging_profile_id="xxxxxxxxxx",
        timeout=10.0,
    ),
    public_key="your-public-key",
)

# Check if a number supports RCS before sending
can_rcs = await provider.check_capability("+15559999999")
if can_rcs:
    rcs = RCSChannel("rcs-main", provider=provider)
```

### RCS Channel Capabilities

RCS channels support richer content than SMS:

```python
# SMS capabilities (for comparison)
ChannelCapabilities(
    media_types=[ChannelMediaType.TEXT, ChannelMediaType.MEDIA],
    max_length=1600,
)

# RCS capabilities
ChannelCapabilities(
    media_types=[ChannelMediaType.TEXT, ChannelMediaType.RICH, ChannelMediaType.MEDIA],
    max_length=8000,
    supports_read_receipts=True,
    supports_typing=True,
    supports_rich_text=True,
    supports_buttons=True,
    supports_quick_replies=True,
    supports_cards=True,
)
```

---

## Unified Webhook Parsing

The `extract_sms_meta()` utility normalizes webhooks across all providers:

```python
from __future__ import annotations

from roomkit.providers.sms import extract_sms_meta

meta = extract_sms_meta("twilio", payload)     # or "telnyx", "sinch", "voicemeup"

if meta.is_inbound:
    inbound = meta.to_inbound(channel_id="sms-main")
    await kit.process_inbound(inbound)
elif meta.is_status:
    status = meta.to_status()
    # Handle delivery status (sent, delivered, failed)
```

**WebhookMeta fields**: `sender`, `recipient`, `body`, `external_id`, `timestamp`, `media_urls`, `direction`, `event_type`, `raw`.

## Phone Number Utilities

```python
from __future__ import annotations

from roomkit.providers.sms import is_valid_phone, normalize_phone

# Normalize to E.164 format
number = normalize_phone("418-555-1234", default_region="CA")
# → "+14185551234"

# Validate
if is_valid_phone("+15145551234"):
    print("Valid!")
```

Requires: `pip install roomkit[phonenumbers]`

---

## Provider Comparison

| Feature | Twilio | Telnyx | Sinch | VoiceMeUp |
|---------|--------|--------|-------|-----------|
| **SMS** | Yes | Yes | Yes | Yes |
| **MMS** | Up to 10 URLs | Multiple URLs | Multiple URLs | 1 attachment |
| **RCS** | Yes (via Messaging Service) | Yes (agent-based) | No | No |
| **Webhook format** | Form-encoded | JSON | JSON | JSON |
| **Signature** | HMAC-SHA1 | ED25519 | HMAC-SHA1 | N/A |
| **Regional endpoints** | No | No | us/eu/au/br/ca | No |
| **Messaging Service** | Yes | Profile-based | Plan-based | No |
| **Sandbox** | Test credentials | N/A | N/A | Yes |
| **Long message split** | Provider-side | Provider-side | Provider-side | Client-side (1000 chars) |
| **MMS aggregation** | Single webhook | Single webhook | Single webhook | Split webhook (auto-merged) |
| **Capability check** | No | RCS only | No | No |

## Testing with MockSMSProvider

```python
from __future__ import annotations

from roomkit.channels import SMSChannel
from roomkit.providers.sms import MockSMSProvider

provider = MockSMSProvider()
sms = SMSChannel("sms-test", provider=provider, from_number="+15550001111")

# ... run your test ...

# Assert messages were sent
assert len(provider.sent) == 1
assert provider.sent[0]["to"] == "+15559999999"
```
