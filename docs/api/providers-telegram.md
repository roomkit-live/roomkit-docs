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

::: roomkit.providers.telegram.bot.TelegramBotProvider

::: roomkit.providers.telegram.config.TelegramConfig

::: roomkit.providers.telegram.mock.MockTelegramProvider

::: roomkit.providers.telegram.webhook.parse_telegram_webhook
