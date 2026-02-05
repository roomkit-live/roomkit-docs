# RoomKit

Pure async Python library for multi-channel conversations.

RoomKit provides the primitives for building multi-channel AI conversation systems: rooms, channels, hooks, identity resolution, and event routing.

## Quick install

```bash
pip install roomkit
```

## Key concepts

- **Room** — A conversation container that holds events, participants, and channel bindings
- **Channel** — A communication endpoint (SMS, Email, WebSocket, AI, etc.)
- **Hook** — Middleware that intercepts events before/after broadcast
- **Event Router** — Distributes events to target channels with access control and transcoding
- **Identity Pipeline** — Resolves unknown senders to known identities
- **Realtime** — Ephemeral events (typing indicators, presence, read receipts)

## Minimal example

```python
from roomkit import RoomKit, TextContent

kit = RoomKit()

# Register a channel, create a room, attach the channel
kit.register_channel(my_channel)
room = await kit.create_room()
await kit.attach_channel(room.id, my_channel.channel_id)

# Process inbound messages
result = await kit.process_inbound(message)
```

## Documentation

- **[Why RoomKit](features.md#why-roomkit)** — Architectural decisions that make RoomKit valuable
- **[Features](features.md)** — Comprehensive guide to all RoomKit features
- **[API Reference](api/index.md)** — Full API documentation
- **[AI Assistant Integration](ai-integration.md)** — llms.txt and AGENTS.md for AI coding tools

## For AI Assistants

RoomKit provides AI-friendly documentation:

- **[llms.txt](llms.txt)** — Structured documentation index for LLMs
- **[AGENTS.md](https://github.com/sboily/roomkit/blob/main/AGENTS.md)** — Project guidance for coding assistants
