# Teams Providers

Microsoft Teams integration via the [Bot Framework SDK](https://github.com/microsoft/botbuilder-python). Supports inbound webhook parsing and proactive outbound messaging through stored conversation references.

## Quick start

```python
from roomkit import (
    RoomKit,
    BotFrameworkTeamsProvider,
    TeamsConfig,
    parse_teams_webhook,
)
from roomkit.channels import TeamsChannel

config = TeamsConfig(
    app_id="YOUR_APP_ID",
    app_password="YOUR_APP_PASSWORD",
    tenant_id="YOUR_TENANT_ID",  # or "common" for multi-tenant
)
provider = BotFrameworkTeamsProvider(config)

kit = RoomKit()
kit.register_channel(TeamsChannel("teams-main", provider=provider))
```

## Configuration

`TeamsConfig` holds the Azure Bot registration credentials:

```python
from roomkit import TeamsConfig

config = TeamsConfig(
    app_id="your-azure-app-id",          # Azure AD application (client) ID
    app_password="your-client-secret",    # Azure AD client secret
    tenant_id="common",                   # "common" for multi-tenant (default)
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `app_id` | `str` | required | Azure AD application (client) ID |
| `app_password` | `SecretStr` | required | Azure AD client secret |
| `tenant_id` | `str` | `"common"` | Azure AD tenant ID (`"common"` for multi-tenant bots) |

!!! note "Single-tenant apps"
    For single-tenant Azure AD apps, you **must** set `tenant_id` to your actual tenant ID (a UUID).
    When `tenant_id` is not `"common"`, the provider automatically passes it as `channel_auth_tenant`
    to the Bot Framework SDK, which is required for token acquisition to succeed.

## Webhook integration

Bot Framework sends one Activity per HTTP POST to your messaging endpoint. Use `parse_teams_webhook()` to convert message activities into `InboundMessage`, and `parse_teams_activity()` / `is_bot_added()` to handle lifecycle events:

```python
from aiohttp import web
from roomkit import is_bot_added, parse_teams_activity, parse_teams_webhook

async def handle_messages(request: web.Request) -> web.Response:
    payload = await request.json()

    # Always save the conversation reference (needed for proactive sends)
    await provider.save_conversation_reference(payload)

    activity = parse_teams_activity(payload)

    # Handle bot installation
    if is_bot_added(payload):
        print(f"Bot added to {activity['conversation_type']} conversation")
        return web.Response(status=200)

    # Handle regular messages
    if activity["activity_type"] == "message":
        messages = parse_teams_webhook(payload, channel_id="teams-main")
        for inbound in messages:
            result = await kit.process_inbound(inbound)

    return web.Response(status=200)
```

**FastAPI variant:**

```python
from fastapi import FastAPI, Request
from roomkit import is_bot_added, parse_teams_activity, parse_teams_webhook

app = FastAPI()

@app.post("/api/messages")
async def handle_messages(request: Request):
    payload = await request.json()
    await provider.save_conversation_reference(payload)

    if is_bot_added(payload):
        return {"status": "bot_added"}

    activity = parse_teams_activity(payload)
    if activity["activity_type"] == "message":
        for inbound in parse_teams_webhook(payload, channel_id="teams-main"):
            await kit.process_inbound(inbound)

    return {"status": "ok"}
```

### Webhook metadata

Each parsed `InboundMessage` includes metadata extracted from the Activity:

| Key | Type | Description |
|-----|------|-------------|
| `sender_name` | `str` | Display name of the sender |
| `conversation_id` | `str` | Teams conversation ID (used as `to` for outbound) |
| `conversation_type` | `str` | `"personal"`, `"groupChat"`, or `"channel"` |
| `is_group` | `bool` | Whether the message is from a group/channel chat |
| `bot_mentioned` | `bool` | Whether the bot was `@mentioned` (always `True` in personal chats) |
| `service_url` | `str` | Bot Framework service URL for this conversation |
| `tenant_id` | `str` | Azure AD tenant ID of the sender |

### Mention stripping

In group chats and channels, users `@mention` the bot to address it. The webhook parser automatically strips `<at>BotName</at>` tags from the message text, so your hooks receive clean text:

```
Raw:    "<at>MyBot</at> what's the weather?"
Parsed: "what's the weather?"
```

### Activity parsing helpers

`parse_teams_activity()` extracts common fields from any Bot Framework Activity (not just messages). Useful for handling lifecycle events like `conversationUpdate`:

```python
from roomkit import parse_teams_activity

activity = parse_teams_activity(payload)
# Returns: {
#   "activity_type": "message" | "conversationUpdate" | ...,
#   "conversation_id": "19:abc123@thread.v2",
#   "conversation_type": "personal" | "groupChat" | "channel",
#   "is_group": True/False,
#   "service_url": "https://smba.trafficmanager.net/amer/",
#   "tenant_id": "your-tenant-id",
#   "sender_id": "...",
#   "sender_name": "...",
#   "bot_id": "...",
#   "members_added": [...],
#   "members_removed": [...],
# }
```

### Bot installation detection

`is_bot_added()` checks if a `conversationUpdate` Activity indicates the bot was added to a conversation:

```python
from roomkit import is_bot_added

if is_bot_added(payload):
    conv_type = parse_teams_activity(payload)["conversation_type"]
    print(f"Bot installed in {conv_type} chat")
```

You can optionally pass a `bot_id` parameter. If not provided, it uses `payload["recipient"]["id"]`.

## Conversation references

Teams requires **conversation references** for proactive (bot-initiated) messaging. When a user messages the bot, you capture the reference from the inbound Activity. Later, you use that reference to send messages back.

`BotFrameworkTeamsProvider` manages this automatically with a pluggable store:

```python
# Capture reference from inbound webhook
await provider.save_conversation_reference(activity_dict)

# Later: send proactively using the conversation ID as `to`
result = await provider.send(event, to="conversation-id-here")
```

### Custom conversation store

The default `InMemoryConversationReferenceStore` works for single-process bots. For production, implement `ConversationReferenceStore` with a persistent backend:

```python
from roomkit import ConversationReferenceStore

class RedisConversationStore(ConversationReferenceStore):
    async def save(self, conversation_id, reference):
        await self.redis.set(f"teams:ref:{conversation_id}", json.dumps(reference))

    async def get(self, conversation_id):
        data = await self.redis.get(f"teams:ref:{conversation_id}")
        return json.loads(data) if data else None

    async def delete(self, conversation_id):
        await self.redis.delete(f"teams:ref:{conversation_id}")

    async def list_all(self):
        # Return all stored references
        ...

provider = BotFrameworkTeamsProvider(
    config,
    conversation_store=RedisConversationStore(redis),
)
```

### Proactive channel messaging

To message a Teams channel the bot has been installed in — even if no user has messaged the bot in that channel yet — use `create_channel_conversation()`:

```python
# Create a conversation in a Teams channel
conv_id = await provider.create_channel_conversation(
    service_url="https://smba.trafficmanager.net/amer/",
    channel_id="19:abc123@thread.tacv2",
    tenant_id="your-tenant-id",  # optional, falls back to config
)

# Now you can send messages using the returned conversation ID
result = await provider.send(event, to=conv_id)
```

The `service_url` and `channel_id` are available from a prior `conversationUpdate` Activity (captured via `parse_teams_activity()`).

## Channel capabilities

| Feature | Supported |
|---------|-----------|
| Text messages | Yes |
| Rich text (HTML) | Yes |
| Threading | Yes |
| Reactions | Yes |
| Read receipts | Yes |
| Max message length | 28,000 characters |
| Media attachments | Not yet |
| Adaptive Cards | Not yet |

## Installation

```bash
pip install roomkit[teams]
```

This installs `botbuilder-core>=4.14`. The `BotFrameworkTeamsProvider` constructor raises a helpful `ImportError` if the dependency is missing.

## Testing with Bot Framework Emulator

You can test locally without Azure credentials using the [Bot Framework Emulator](https://github.com/microsoft/BotFramework-Emulator):

1. Leave `app_id` and `app_password` empty for local testing
2. Run your bot on `http://localhost:3978/api/messages`
3. Connect the Emulator to that URL

```python
# Local testing config (no auth)
config = TeamsConfig(app_id="", app_password="")
provider = BotFrameworkTeamsProvider(config)
```

## Testing with MockTeamsProvider

For unit tests, use `MockTeamsProvider` which records all sent messages:

```python
from roomkit import MockTeamsProvider

mock = MockTeamsProvider()
result = await mock.send(event, to="conv-123")

assert result.success
assert len(mock.sent) == 1
assert mock.sent[0]["to"] == "conv-123"
```

## API reference

::: roomkit.TeamsProvider

::: roomkit.BotFrameworkTeamsProvider

::: roomkit.MockTeamsProvider

::: roomkit.TeamsConfig

::: roomkit.parse_teams_webhook

::: roomkit.parse_teams_activity

::: roomkit.is_bot_added

::: roomkit.ConversationReferenceStore

::: roomkit.InMemoryConversationReferenceStore
