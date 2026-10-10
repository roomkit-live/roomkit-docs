# RoomKit FAQ

Common questions about RoomKit's architecture, scope, and integration patterns.

---

## Vision & Philosophy

### What is RoomKit?

RoomKit is a **room-centric conversation orchestration library**. It provides abstractions for managing multi-channel conversations (SMS, Email, WebSocket, AI, etc.) within the concept of "rooms" — containers that hold events, participants, and channel bindings.

### What problem does RoomKit solve?

RoomKit solves the complexity of building applications where conversations span multiple channels. Instead of building separate integrations for SMS, email, chat, and AI, you get:

- **Unified event model**: All messages become `RoomEvent`s regardless of source
- **Channel abstraction**: Swap providers (Twilio → Telnyx) without changing application code
- **Routing & hooks**: Control how messages flow between channels
- **Identity resolution**: Link anonymous senders to known identities

### What is RoomKit NOT?

RoomKit is **not** a complete chat backend. It doesn't handle:

- User authentication or session management
- Deciding which rooms a user may open
- Push notification infrastructure
- Unread badges, and what counts as "read"
- User presence across your application

These are intentionally left to integrators because they vary significantly between applications.

---

## Architecture Questions

### Can one WebSocket carry several rooms?

**Yes.** `WebSocketChannel` delivers a room's events to the connections
registered for that room, and one connection can follow several rooms:

```python
from roomkit import RoomEvent, WebSocketChannel

ws = WebSocketChannel("ws-app")
kit.register_channel(ws)
await kit.attach_channel("room-1", "ws-app")
await kit.attach_channel("room-2", "ws-app")

async def send(connection_id: str, event: RoomEvent) -> None:
    # event.room_id tells the client which conversation the event belongs to
    await sockets[connection_id].send_json(event.model_dump(mode="json"))

ws.register_connection("conn-alice", send, room_id="room-1")
ws.subscribe("conn-alice", "room-2")    # the client opens a second conversation
ws.unsubscribe("conn-alice", "room-1")  # ...and closes the first one
```

`ws.rooms_for(connection_id)` lists the rooms a connection follows, and
`ws.unregister_connection(connection_id)` drops it when the socket closes.

What stays in your application, above RoomKit:

```
┌─────────────────────────────────────────┐
│  Your application                       │
│  - Who the user behind a socket is      │
│  - Which rooms that user may open       │
│  - Unread badges & notifications        │
│  - Presence across the application      │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  RoomKit                                │
│  - Rooms, events, read positions        │
│  - Channels and delivery                │
│  - Hooks & routing                      │
└─────────────────────────────────────────┘
```

**Other building blocks:**

- `kit.get_timeline(room_id)` — a room's history
- `AFTER_BROADCAST` hooks — react to every stored event, for notifications or
  your own fan-out
- `kit.subscribe_room(room_id, callback)` — the room's ephemeral events only
  (typing, presence, read receipts), not its messages

### Does RoomKit track unread messages?

It keeps a **read position per channel** in each room:
`kit.mark_read(room_id, channel_id, event_id)`,
`kit.mark_all_read(room_id, channel_id)`, `kit.list_read_markers(room_id)`,
and `kit.store.get_unread_count(room_id, channel_id)`.

What that position means is your application's decision:

- What counts as "read"? Opening the room? Scrolling past the message?
- Do you count every message, or only @mentions? System messages?
- One position per user, or per device? RoomKit keeps one per channel
  binding.

---

## Channel & Provider Questions

### What's the difference between a Channel and a Provider?

- **Provider**: Handles the actual sending/receiving (e.g., `TwilioSMSProvider`, `AnthropicAIProvider`)
- **Channel**: Wraps a provider with RoomKit integration (routing, hooks, metadata)

```python
# Provider = how to send
provider = TwilioSMSProvider(config)

# Channel = RoomKit integration
channel = SMSChannel("sms-main", provider=provider, from_number="+1...")

# Register with RoomKit
kit.register_channel(channel)
```

### Can I have multiple channels of the same type?

Yes! This is a core feature. You might have:

```python
kit.register_channel(SMSChannel("sms-twilio", provider=twilio_provider))
kit.register_channel(SMSChannel("sms-telnyx", provider=telnyx_provider))
kit.register_channel(SMSChannel("sms-sinch", provider=sinch_provider))
```

Each channel has a unique ID and can be attached to rooms independently.

### What's a SourceProvider vs a regular Provider?

- **Provider** (outbound): RoomKit calls it to send a message out, when the
  channel it backs delivers an event.
- **SourceProvider** (inbound): holds a connection open (a WebSocket, a
  Server-Sent Events stream, a WhatsApp session…) and pushes what arrives into
  RoomKit. Attach it with `kit.attach_source(channel_id, source)`.

For **bidirectional** traffic over one connection, the channel's provider sends
through the source. WhatsApp Personal works this way:

```python
from roomkit import RoomKit, WhatsAppPersonalChannel
from roomkit.providers.whatsapp.personal import WhatsAppPersonalProvider
from roomkit.sources import WhatsAppPersonalSourceProvider

kit = RoomKit()

source = WhatsAppPersonalSourceProvider(db="wa-session.db", channel_id="wa-personal")
provider = WhatsAppPersonalProvider(source)  # outbound goes through the source's session

kit.register_channel(WhatsAppPersonalChannel("wa-personal", provider=provider))
await kit.attach_source("wa-personal", source, auto_restart=True)
```

See [Sources](api/sources.md#bidirectional-pattern-source-provider) for the
other sources and their options.

---

## Hooks & Routing Questions

### When should I use hooks vs custom logic?

**Use hooks when:**
- You need to intercept/modify events in the pipeline
- Logic applies to specific channel types or directions
- You want RoomKit to manage the lifecycle

**Use custom logic when:**
- You need access to external services/state
- Logic is application-specific (not conversation-related)
- You're building on top of RoomKit events (fan-out, notifications)

### What's the difference between BEFORE_BROADCAST and AFTER_BROADCAST?

- **BEFORE_BROADCAST**: Event is created but not yet stored/delivered. You can modify or block it.
- **AFTER_BROADCAST**: Event is stored and delivered. Use for side effects (notifications, logging, analytics).

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def redact_sensitive(event, ctx):
    if contains_pii(event.content):
        return HookResult.block("PII detected")
    return HookResult.allow()

@kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC)
async def notify_external(event, ctx):
    # Runs after delivery; its return value is ignored
    await send_to_analytics(event)
```

---

## Identity & Participants Questions

### What's the difference between a Participant and a User?

- **Participant**: Someone in a specific room conversation (may be anonymous, pending identification)
- **User**: An authenticated entity in your application (RoomKit doesn't manage this)

A single user might be multiple participants across different rooms. A participant might not be a user at all (e.g., someone texting from an unknown number).

### When does identity resolution happen?

Identity resolution runs when:
1. An inbound message arrives from a channel with identity resolution enabled
2. The sender's address (phone, email) is looked up against known identities
3. Based on the result (identified, ambiguous, unknown), identity hooks can
   intervene

```python
from roomkit import ChannelType, HookTrigger, RoomKit
from roomkit.models import IdentityHookResult

kit = RoomKit(
    identity_resolver=your_resolver,
    identity_channel_types={ChannelType.SMS, ChannelType.EMAIL},
)

@kit.identity_hook(HookTrigger.ON_IDENTITY_AMBIGUOUS)
async def handle_ambiguous(event, ctx, id_result):
    # Several identities match this phone number: keep the sender pending
    # until someone picks one with kit.resolve_participant()
    return IdentityHookResult.pending(candidates=id_result.candidates)
```

---

## Scaling & Production Questions

### How does RoomKit handle persistence?

RoomKit uses a **pluggable store** abstraction. The default `InMemoryStore` is
ephemeral. For persistent deployments, RoomKit ships two backends:

```python
from roomkit import RoomKit, SQLiteStore
from roomkit.store import PostgresAdvisoryLockManager, PostgresStore

# Embedded persistence for one process, no optional dependency.
embedded = RoomKit(store=SQLiteStore("roomkit.db"))

# Shared persistence for horizontally scaled workers.
postgres = PostgresStore("postgresql://user:pass@db/roomkit")
await postgres.init()
locks = PostgresAdvisoryLockManager(dsn="postgresql://user:pass@db/roomkit")
await locks.init()
distributed = RoomKit(store=postgres, lock_manager=locks)
```

SQLite must remain single-process at the RoomKit level; database write locking
does not serialize the complete inbound pipeline. See the
[SQLite guide](guides/sqlite-store.md), [PostgreSQL guide](guides/postgres-store.md),
and [Store API](api/store.md).

### Can RoomKit scale horizontally?

Yes, with the right store backend. For multi-instance deployments:

1. Share one `PostgresStore`, with `PostgresAdvisoryLockManager` (below)
2. Share ephemeral events (typing, presence) through `RedisRealtimeBackend`
   from `roomkit.realtime` (`roomkit[redis]`)
3. Coordinate source connections yourself (one instance per source)

### How do I handle provider rate limits?

Configure retry policies and rate limits when attaching channels:

```python
from roomkit.models.channel import RetryPolicy, RateLimit

await kit.attach_channel(
    room_id,
    channel_id,
    retry_policy=RetryPolicy(max_retries=3, base_delay_seconds=1.0),
    rate_limit=RateLimit(max_per_second=10),
)
```

Or implement rate limiting in hooks:

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def rate_limit(event, ctx):
    if await is_rate_limited(event.source.channel_id):
        return HookResult.block("Rate limited - try again later")
    return HookResult.allow()
```

### How do I run multiple instances (horizontal scaling)?

When several processes share one persistent store, room processing must be
serialized *across* processes. RoomKit **ships** a PostgreSQL advisory-lock
manager for exactly this — pair it with `PostgresStore`:

```python
from roomkit import RoomKit
from roomkit.store import PostgresStore, PostgresAdvisoryLockManager

store = PostgresStore(dsn="postgresql://user:pass@db/roomkit")
await store.init()

# Give the lock manager its OWN pool (separate from the store's query pool): a
# session advisory lock is held on its connection for the whole locked section,
# so sharing the store's pool could deadlock.
locks = PostgresAdvisoryLockManager(
    dsn="postgresql://user:pass@db/roomkit",
    max_size=20,   # ≈ rooms processed concurrently per process
)
await locks.init()

kit = RoomKit(store=store, lock_manager=locks)
```

The manager maps each room id to an advisory key with a **stable** hash
(`blake2b`), never Python's built-in `hash()` — which is randomized per process
(`PYTHONHASHSEED`), so two workers would derive different keys for the same room
and fail to serialize. As defence in depth, index assignment is authoritative at
the storage layer regardless of the lock: `commit_event` assigns the index and
bumps the room counters in a single transaction, backed by a
`UNIQUE(room_id, index)` constraint — so a misconfigured lock surfaces as a loud
constraint error, never silent corruption.

To back the lock with another system (Redis, etcd, …), implement
`RoomLockManager` yourself — but keep the same guarantees: a **stable** key, and
lock/unlock on the **same** connection.

```python
from contextlib import asynccontextmanager

from roomkit import RoomLockManager

class RedisLockManager(RoomLockManager):
    def __init__(self, redis_client):
        self._redis = redis_client

    @asynccontextmanager
    async def locked(self, room_id: str):
        async with self._redis.lock(f"roomkit:room:{room_id}", timeout=30):
            yield
```

See the [Horizontal Scaling guide](guides/scaling.md) for pool sizing and the
startup warning.

---

## Testing Questions

### How do I test with RoomKit?

The default `InMemoryStore` and the mock providers make no network call. A
`WebSocketChannel` with a collecting callback stands in for the user:

```python
from roomkit import AIChannel, InboundMessage, RoomEvent, RoomKit, TextContent, WebSocketChannel
from roomkit.providers.ai.mock import MockAIProvider


async def test_ai_answers() -> None:
    kit = RoomKit()  # InMemoryStore by default
    user = WebSocketChannel("ws-user")
    ai = AIChannel("ai-test", provider=MockAIProvider(responses=["Hello!"]))
    kit.register_channel(user)
    kit.register_channel(ai)

    await kit.create_room(room_id="test-room")
    await kit.attach_channel("test-room", "ws-user")
    await kit.attach_channel("test-room", "ai-test")

    received: list[RoomEvent] = []

    async def collect(connection_id: str, event: RoomEvent) -> None:
        received.append(event)

    user.register_connection("conn-1", collect, room_id="test-room")

    result = await kit.process_inbound(
        InboundMessage(channel_id="ws-user", sender_id="test-user", content=TextContent(body="Hi"))
    )

    assert not result.blocked
    assert [e.content.body for e in received] == ["Hello!"]
```

See [Testing Patterns](guides/testing-patterns.md) for voice, providers and
hooks.

### How do I test hooks?

Hooks run during normal message processing, so test them end-to-end, on the
same setup:

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST, name="spam_filter")
async def block_spam(event, ctx):
    if isinstance(event.content, TextContent) and "spam" in event.content.body.lower():
        return HookResult.block("Spam detected")
    return HookResult.allow()

result = await kit.process_inbound(
    InboundMessage(channel_id="ws-user", sender_id="test-user", content=TextContent(body="Buy spam now"))
)
assert result.blocked
```

---

## Getting Help

- **GitHub Issues**: Bug reports and feature requests
- **Discussions**: Architecture questions and patterns
- **Examples**: See `/examples` for common integration patterns
