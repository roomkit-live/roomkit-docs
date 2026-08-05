# Telegram Provider

Install with: `pip install roomkit[telegram]` (bundles `telegramify-markdown`,
used to render Markdown into Telegram entities).

## Markdown & Rich Messages

The provider renders outbound Markdown into native Telegram formatting via
`telegramify-markdown`. By default it sends text with **message entities**
(bold, code, links, etc.); if the converter is unavailable or conversion fails,
it falls back to sending the raw text unformatted.

Set `rich_messages=True` on `TelegramConfig` to opt in to **Bot API 10.1 Rich
Messages** — native tables and headings — for text sends. Each send tries the
Rich Message path first and falls back to entity formatting on any failure (and
never re-sends content that already reached the chat):

```python
from roomkit.providers.telegram.config import TelegramConfig

config = TelegramConfig(
    bot_token="...",
    rich_messages=True,   # native tables/headings (Bot API 10.1)
)
```

## Two parsing layers

`parse_telegram_message(msg)` reads a Telegram `message` object into its parts
and attributes nothing:

```python
parts = parse_telegram_message(update["message"])   # TelegramMessageParts | None
parts.content        # TextContent (caption for media) | LocationContent
parts.metadata       # chat_id, date, and the media keys below
parts.message_id
parts.sender_id      # offered, not imposed
```

`parse_telegram_webhook` is that function plus the ordinary attribution — the
sender is `message.from.id` — wrapped in an `InboundMessage`.

Reach for the lower layer when your identity model is not Telegram's. Under a
one-bot-per-user deployment a direct message belongs to the bot's *owner*, not
to the account that typed it, and the same process may apply the opposite rule
in a group. Reading a media file's `file_id` should not cost a consumer its
identity model, which is why the layer is public.

## Inbound media

`parse_telegram_webhook` handles `photo`, `voice`, `audio`, `video_note`,
`video` and `document`. Each parses to a `TextContent` whose body is the
caption — empty when there is none, which is the normal case for a voice note —
carrying the file reference in metadata:

| Key | Always | Notes |
|---|---|---|
| `file_id` | yes | pass to `get_file()` |
| `media_type` | yes | `voice`, `audio`, `video_note`, `video`, `document`, `photo` |
| `duration` | no | audio and video kinds |
| `mime_type` | no | absent on `video_note` |
| `file_name` | no | `audio`, `video`, `document` |
| `file_size` | no | bytes |

A `file_id` is only useful to whoever holds the bot token, which is the
provider — so the two resolution steps live there:

```python
file_id = inbound.metadata["file_id"]

file_path = await provider.get_file(file_id)        # getFile → path, valid ≥ 1 hour
if file_path:
    data = await provider.download_file(file_path)  # bytes
```

Both return `None` on failure and log a warning that never contains the URL —
every Bot API URL embeds the bot token. Telegram caps Bot API downloads at
**20 MB** and refuses larger files at the `getFile` step, so `get_file()`
returns `None` for them; `metadata["file_size"]` lets you tell before spending
the call.

RoomKit stops at the bytes. Transcribing a voice note, or storing an
attachment, is the application's choice of engine and policy.

::: roomkit.providers.telegram.bot.TelegramBotProvider

::: roomkit.providers.telegram.config.TelegramConfig

::: roomkit.providers.telegram.mock.MockTelegramProvider

::: roomkit.providers.telegram.webhook.parse_telegram_webhook

::: roomkit.providers.telegram.webhook.parse_telegram_message

::: roomkit.providers.telegram.webhook.TelegramMessageParts
