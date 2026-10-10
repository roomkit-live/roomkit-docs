# RoomKit

Pure async Python library for multi-channel conversations.

A **room** holds one conversation. **Channels** connect it to the outside
world: SMS, MMS, RCS, email, WhatsApp, Messenger, Teams, Telegram, Discord,
WebSocket, HTTP webhooks, voice, video and AI. A message that enters on one
channel is stored in the room, then delivered to every other channel attached
to it, adapted to what each one can display. **Hooks** sit in that path to
block, modify or observe it.

## Install

```bash
pip install roomkit
```

Providers with third-party SDKs come as extras, for example
`pip install "roomkit[anthropic]"`, `"roomkit[postgres]"` or `"roomkit[mcp]"`.
The library needs Python 3.12 or later.

## Minimal example

A browser socket and an AI assistant in one room:

```python
import asyncio

from roomkit import (
    AIChannel,
    InboundMessage,
    RoomEvent,
    RoomKit,
    TextContent,
    WebSocketChannel,
)
from roomkit.providers.ai.mock import MockAIProvider


async def main() -> None:
    kit = RoomKit()

    # A browser connection, and an AI assistant (a mock here, a real provider in production)
    browser = WebSocketChannel("ws-browser")
    assistant = AIChannel("ai-assistant", provider=MockAIProvider(responses=["Hi! How can I help?"]))
    kit.register_channel(browser)
    kit.register_channel(assistant)

    # One room, both channels attached
    room = await kit.create_room()
    await kit.attach_channel(room.id, "ws-browser")
    await kit.attach_channel(room.id, "ai-assistant")

    # What the room delivers to the browser socket
    async def send_to_browser(connection_id: str, event: RoomEvent) -> None:
        print(f"browser <- {event.content.body}")

    browser.register_connection("conn-1", send_to_browser, room_id=room.id)

    # A message from the browser enters the room; the assistant's answer comes back
    await kit.process_inbound(
        InboundMessage(channel_id="ws-browser", sender_id="alice", content=TextContent(body="Hello"))
    )

    await kit.close()


asyncio.run(main())
```

It prints `browser <- Hi! How can I help?`. Replace the mock with
`AnthropicAIProvider`, `OpenAIAIProvider` or another provider, and the socket
with `SMSChannel` or any other channel: the room code does not change.

## Key concepts

| Concept | What it does |
|---|---|
| **Room** | Holds a conversation's events, participants and channel bindings. Events are numbered in order within a room. |
| **Channel** | Connects a room to one medium. A transport channel wraps a **provider** (Twilio, Telnyx, SendGrid…), so the vendor can change without touching the room. |
| **Binding** | Attaches a channel to a room, with its access, mute and visibility settings. |
| **Hook** | Runs at a point of the pipeline: before broadcast (can block or modify) or after it (side effects). |
| **Store** | Keeps rooms and events: in memory by default, SQLite or PostgreSQL for persistence. |
| **Identity** | Resolves an address (a phone number, an email) to a known person, and can verify them. |
| **Realtime** | Ephemeral events that are not stored: typing, presence, read receipts, tool progress. |

Beyond text, RoomKit covers voice (STT, TTS and an audio pipeline, or
speech-to-speech models), video, conferences, and several AI agents working in
one room.

## Where to go next

- **[Features](features.md)**: every feature, starting with [why RoomKit](features.md#why-roomkit)
- **[Architecture](architecture.md)** and **[Technical](technical.md)**: how the pieces fit
- **Guides** by topic: [tool calling](guides/tool-calling.md),
  [voice interruption](guides/voice-interruption.md),
  [realtime voice](guides/realtime-voice-providers.md),
  [video](guides/video-overview.md),
  [multi-agent orchestration](guides/orchestration.md),
  [identity](guides/identity-resolution.md),
  [PostgreSQL store](guides/postgres-store.md),
  [testing](guides/testing-patterns.md)
- **[API Reference](api/index.md)**: generated from the docstrings
- **[FAQ](faq.md)**: scope and integration questions
- **[AI Assistant Integration](ai-integration.md)**: `llms.txt`, `AGENTS.md` and
  Agent Skills for coding assistants
