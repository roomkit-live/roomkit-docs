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
parts.content              # TextContent (caption for media) | LocationContent
parts.metadata             # chat_id, date, and the media keys below
parts.message_id
parts.sender_id            # offered, not imposed
parts.entities             # message entities, or caption_entities for media
parts.reply_to_message_id  # what a reply answers, or None
parts.media_group_id       # the album this belongs to, or None
```

The last three are protocol facts, not attribution: they stay off the
`InboundMessage`, so `parse_telegram_webhook`'s metadata is unchanged.

`parse_telegram_webhook` is that function plus the ordinary attribution — the
sender is `message.from.id` — wrapped in an `InboundMessage`. Its `external_id`
and `idempotency_key` are `<chat_id>:<message_id>`: Telegram reuses message ids
between chats. Malformed nested objects, missing ids and invalid coordinates
are rejected with no message rather than escaping an exception from a webhook
boundary.

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
every Bot API URL embeds the bot token. The same rule applies to every API
transport error: its safe exception class is returned, never `str(exc)` with
the token-bearing URL. Telegram caps Bot API downloads at
**20 MB** and refuses larger files at the `getFile` step, so `get_file()`
returns `None` for them; `metadata["file_size"]` lets you tell before spending
the call.

RoomKit stops at the bytes. Transcribing a voice note, or storing an
attachment, is the application's choice of engine and policy.

## The Bot API surface

`TelegramBotProvider` is `TelegramBotAPI` plus the rendering of a `RoomEvent`.
The API half is what an application needs *around* its sends — registering a
webhook, identifying its own bot, acknowledging a button press, rewriting a
message it already sent — so it never writes a second HTTP client for a token
the provider already holds.

```python
import os

config = TelegramConfig(
    bot_token="...",
    # Generate this once and persist it in a secret manager.
    webhook_secret=os.environ["TELEGRAM_WEBHOOK_SECRET"],
)
provider = TelegramBotProvider(config)

me = await provider.get_me()
if not me.success:
    ...                              # me.error, me.metadata["description"]
bot = me.metadata["result"]          # id, username, first_name

await provider.set_webhook(
    "https://example.com/hooks/telegram",
    allowed_updates=["message", "callback_query"],       # ask for what you need
)

# In the HTTP endpoint, before parsing or processing the update:
raw = await request.body()
secret_header = request.headers.get("X-Telegram-Bot-Api-Secret-Token", "")
if not provider.verify_signature(raw, secret_header):
    raise HTTPException(status_code=403)
```

`set_webhook()` uses `TelegramConfig.webhook_secret` when `secret` is omitted.
If an explicit secret is supplied to rotate it, the provider starts verifying
that value only after Telegram accepts the registration; registration and
verification therefore keep one runtime source of truth.

| Call | Bot API | Notes |
|---|---|---|
| `get_me()` | `getMe` | bot object under `metadata["result"]` |
| `get_updates(limit, offset)` | `getUpdates` | list under `metadata["result"]`; silent while a webhook is set |
| `set_webhook(url, secret, allowed_updates, drop_pending_updates)` | `setWebhook` | |
| `delete_webhook(drop_pending_updates)` | `deleteWebhook` | |
| `leave_chat(chat_id)` | `leaveChat` | the only way to stop one chat's updates |
| `send_message(chat_id, text)` | `sendMessage` | plain text, no Markdown pass |
| `send_force_reply(chat_id, text)` | `sendMessage` | `provider_message_id` matches the later reply |
| `send_chat_action(chat_id, action)` | `sendChatAction` | Telegram clears it after ~5s |
| `answer_callback_query(id, text)` | `answerCallbackQuery` | required, or the button keeps spinning |
| `edit_message_text(chat_id, message_id, text, reply_markup)` | `editMessageText` | `{"inline_keyboard": []}` drops the buttons |
| `edit_message_reply_markup(chat_id, message_id, reply_markup)` | `editMessageReplyMarkup` | keyboard only, text untouched |

Every one answers with a `ProviderResult`, so failure reads the same way
whichever call produced it: `telegram_<code>` when Telegram refused,
`http_<status>` when the refusal carried no Bot API body, `timeout` when
nothing came back, or a safe transport exception class — with Telegram's own
words under `metadata["description"]`, which is the only text precise enough to
say *why* a webhook URL was rejected. The raw transport exception is never
returned because it can contain the token-bearing request URL.

A successful envelope is also checked against the method's result contract:
`getMe` must return a bot object, `getUpdates` a list of updates with integer
ids, boolean mutations the literal `true`, and sends/edits a Message carrying
an integer `message_id`. A mismatched but otherwise valid JSON payload returns
`invalid_response` instead of leaking a wrong shape to the caller.

## Update forms

```python
update = parse_telegram_update(payload)     # TelegramUpdate | None
if update is None:
    return                                  # a form you did not ask for
if update.callback:
    press = update.callback                 # TelegramCallback
else:
    parts = parse_telegram_message(update.message)
```

`update.edited` tells a new message from an edit of one already delivered —
whether that is a new turn or something to ignore is the application's call.

`TelegramCallback` carries the query `id` to answer, the `data`, the
`sender_id`, and the message the button hangs off (`chat_id`, `message_id`,
`message_text`) so an outcome can be appended to what was already said. Note
that `callback_data` is *posted by whoever pressed the button* — any client can
send arbitrary bytes to its own bot's webhook — so whatever it names is a claim
to check against your own records, never a fact.

## Being addressed, and UTF-16

```python
if mentions_bot(msg, bot_username="luge_bot", bot_id=42):
    ...
```

True on any of the five ways Telegram lets someone reach a bot in a busy room:
a reply to the bot, an unqualified `bot_command` (or one qualified with this
bot's exact `@username`), a `mention` entity, a `text_mention` naming its id, or
the handle posted as plain text with no entity at all. A command qualified for
another bot remains false even when Telegram delivers it to this bot, and a
longer username does not match by prefix. The helper reports the fact and
decides no policy — whether a given group answers only when addressed is your
rule.

`entity_text(text, entity)` slices the stretch an entity covers. Telegram's
offsets count **UTF-16 code units** and Python indexes code points, so
`text[offset:offset + length]` is correct until a character outside the Basic
Multilingual Plane appears earlier in the message — one emoji is enough — and
silently wrong after.

::: roomkit.providers.telegram.bot.TelegramBotProvider

::: roomkit.providers.telegram.api.TelegramBotAPI

::: roomkit.providers.telegram.config.TelegramConfig

::: roomkit.providers.telegram.mock.MockTelegramProvider

::: roomkit.providers.telegram.webhook.parse_telegram_webhook

::: roomkit.providers.telegram.webhook.parse_telegram_message

::: roomkit.providers.telegram.webhook.TelegramMessageParts

::: roomkit.providers.telegram.update.parse_telegram_update

::: roomkit.providers.telegram.update.parse_telegram_callback

::: roomkit.providers.telegram.update.TelegramUpdate

::: roomkit.providers.telegram.update.TelegramCallback

::: roomkit.providers.telegram.entities.mentions_bot

::: roomkit.providers.telegram.entities.entity_text
