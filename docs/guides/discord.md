# Discord Bot

RoomKit talks to Discord through a **bot** that holds a persistent
connection to the Discord **gateway**. Unlike webhook-based channels (Teams,
Telegram), a Discord bot receives messages over a long-lived WebSocket and
sends replies over the REST API — both through a single `discord.Client`.

That single connection is why Discord is wired as a **source + provider pair**
sharing one client, mirroring the WhatsApp-personal integration:

- [`DiscordGatewaySource`][roomkit.sources.discord.DiscordGatewaySource] owns
  the `discord.Client`, receives messages, and emits them into RoomKit.
- [`DiscordBotProvider`][roomkit.providers.discord.bot.DiscordBotProvider]
  reuses that client to send messages back out.

## Install

```bash
pip install roomkit[discord]
```

## Create the bot

In the [Discord Developer Portal](https://discord.com/developers/applications):

1. **New Application** → add a **Bot** → copy its **token**.
2. Under **Bot → Privileged Gateway Intents**, enable the
   **Message Content Intent**. Without it, every inbound `message.content`
   arrives empty.
3. Under **Bot**, make sure **Requires OAuth2 Code Grant** is **OFF**
   (when ON, the invite below fails with *"Integration requires code grant"*).
4. Invite the bot by opening this URL — replace `APP_ID` with your
   **Application ID** (from *General Information*). No redirect URL is needed:

   ```
   https://discord.com/api/oauth2/authorize?client_id=APP_ID&scope=bot&permissions=68672
   ```

   `permissions=68672` = View Channels + Send Messages + Read Message History +
   Add Reactions. Pick a server you manage from the **Add to Server** dropdown
   and click **Authorize**. (You can also build this URL via
   **OAuth2 → URL Generator**, scope `bot`.)

On startup the source logs the servers it joined
(`Discord gateway ready as <bot> in N server(s): ...`) — a quick way to confirm
the invite worked.

## Wire it up

```python
from roomkit import DiscordChannel, RoomKit
from roomkit.providers.discord import DiscordBotProvider, DiscordConfig
from roomkit.sources.discord import DiscordGatewaySource

config = DiscordConfig(bot_token="YOUR_BOT_TOKEN")
source = DiscordGatewaySource(config, channel_id="discord-main")
provider = DiscordBotProvider(source)        # reuses the source's client

kit = RoomKit()
kit.register_channel(DiscordChannel("discord-main", provider=provider))
await kit.attach_source("discord-main", source)   # opens the gateway
```

The recipient key `discord_channel_id` resolves the target channel snowflake
at delivery time: set `binding.metadata["discord_channel_id"]` on the room
binding, or pass a channel id straight to `provider.send(event, to=...)`.

## Inbound messages

Each Discord message is parsed by
[`parse_discord_message`][roomkit.sources.discord.parse_discord_message] into a
RoomKit `InboundMessage`:

- **Text** → `TextContent`.
- **Attachments** → `MediaContent` (the first attachment, with the message
  text as its caption; extra attachment URLs land in
  `metadata["attachment_urls"]`).
- **Replies** → the referenced message id is carried as `thread_id`.

The bot's own messages are always dropped (no echo loop), and other bots are
dropped by default (`DiscordConfig.ignore_bots=True`). Metadata includes
`guild_id`, `channel_id`, `channel_name`, `author_name`, `author_bot`, and
`message_id`.

## Outbound messages

`provider.send(event, to=channel_id)` maps `RoomEvent.content`:

| Content        | Discord output                                             |
| -------------- | ---------------------------------------------------------- |
| `TextContent`  | plain message                                              |
| `RichContent`  | an embed (`description` = `plain_text` or body)            |
| `MediaContent` | http(s) URL in the message (auto-embedded); `data:` URIs are uploaded as a file |

When `event.channel_data.thread_id` is set, the message is sent as a **reply**
to that message id.

## Reactions

Reactions are surfaced as lifecycle events through the source's `on_event`
callback (they are **not** pushed through the message pipeline), and sent with
`provider.send_reaction(...)`:

```python
async def on_discord_event(reaction: dict[str, str]) -> None:
    # {"action": "add"|"remove", "emoji", "user_id", "message_id", "channel_id"}
    print(reaction)

source = DiscordGatewaySource(config, channel_id="discord-main", on_event=on_discord_event)

# outbound
await provider.send_reaction(channel_id="123...", message_id="456...", emoji="🔥")
```

## Capabilities & limits

`DiscordChannel` advertises text, rich (embeds), and media, with threading and
reactions. Max message length is 2000 characters.

Out of scope for now: interactive components (buttons/Views), slash commands,
and voice. Multiple attachments collapse to the first one (the rest are
recorded in metadata).

## Runnable example

See [`examples/discord_bot.py`](https://github.com/roomkit/roomkit/blob/main/examples/discord_bot.py)
for an end-to-end echo bot:

```bash
DISCORD_BOT_TOKEN=... uv run python examples/discord_bot.py
```
