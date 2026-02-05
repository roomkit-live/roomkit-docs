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
- WebSocket connection pooling at the user level
- Push notification infrastructure
- Read receipts / unread count persistence
- User presence across your application

These are intentionally left to integrators because they vary significantly between applications.

---

## Architecture Questions

### Should RoomKit manage user-level WebSocket connections?

**No.** RoomKit is room-centric, not user-centric.

A common pattern is wanting a single WebSocket per user that subscribes to multiple rooms:

```
/ws (one connection per user)
Client → Server: { "action": "subscribe", "room_id": "xxx" }
Server → Client: { "type": "event", "room_id": "xxx", ... }
```

This is **integrator-side responsibility**. Your WebSocket layer sits above RoomKit:

```
┌─────────────────────────────────────────┐
│  Your App's User Session Layer          │
│  - User auth & connection management    │
│  - Room subscription tracking           │
│  - Unread counts & notifications        │
│  - Fan-out logic                        │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  RoomKit                                │
│  - Room/event management                │
│  - Channel abstraction                  │
│  - Hooks & routing                      │
└─────────────────────────────────────────┘
```

**Building blocks RoomKit provides:**
- `kit.subscribe_room(room_id, callback)` — receive events for any room
- `kit.get_timeline(room_id)` — fetch history
- `AFTER_BROADCAST` hooks — trigger your fan-out logic
- Ephemeral events — typing indicators, presence, custom events

### Why doesn't RoomKit handle unread counts?

Unread counts are **application-specific business logic**:

- What counts as "read"? Viewing the room? Scrolling past the message?
- Do you count all messages or just @mentions?
- Are system messages counted?
- Per-user or per-device tracking?

RoomKit provides the events; you decide what "unread" means for your app.

### Should I use RoomKit's WebSocketChannel for my chat UI?

**It depends on your architecture.**

`WebSocketChannel` is designed for:
- Browser connections to specific rooms
- Real-time event delivery within a room context
- Typing indicators and presence within a room

If you need user-level connections (one socket per user, multiple rooms), build your own WebSocket handler that uses RoomKit's primitives:

```python
@app.websocket("/ws")
async def user_ws(ws, user_id):
    subscriptions = {}

    async def on_event(room_id, event):
        await ws.send_json({"room_id": room_id, "event": serialize(event)})

    async for msg in ws:
        if msg["action"] == "subscribe":
            sub_id = await kit.subscribe_room(msg["room_id"],
                lambda e: on_event(msg["room_id"], e))
            subscriptions[msg["room_id"]] = sub_id
```

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
kit.register_channel(SMSChannel("sms-vonage", provider=vonage_provider))
```

Each channel has a unique ID and can be attached to rooms independently.

### What's a SourceProvider vs a regular Provider?

- **Provider** (outbound): Sends messages from RoomKit to external systems
- **SourceProvider** (inbound): Receives messages from external systems into RoomKit

Example: `WebSocketSource` connects to an external WebSocket server and routes incoming messages to RoomKit rooms.

For **bidirectional** communication, pair a Source with a Provider that share the same connection:

```python
from roomkit import Provider, ProviderCapability
from roomkit.sources import WebSocketSource

# Source handles inbound messages
source = WebSocketSource(url="wss://external.example.com/events", channel_id="ws-ext")

# Provider wraps source.send() for outbound messages
class WebSocketProvider(Provider):
    def __init__(self, source: WebSocketSource):
        self._source = source

    async def send(self, message) -> DeliveryResult:
        await self._source.send(serialize(message))
        return DeliveryResult(success=True)

# Register both
provider = WebSocketProvider(source)
kit.register_provider("ws-ext", provider)
await kit.attach_source("ws-ext", source)
```

See the [Bidirectional Channel Pattern](api/sources.md#bidirectional-channel-pattern) documentation for a complete example.

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

@kit.hook(HookTrigger.AFTER_BROADCAST)
async def notify_external(event, ctx):
    await send_to_analytics(event)
    return HookResult.allow()
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
3. Based on the result (identified, ambiguous, unknown), hooks can intervene

```python
kit = RoomKit(
    identity_resolver=your_resolver,
    identity_channel_types={ChannelType.SMS, ChannelType.EMAIL},
)

@kit.hook(HookTrigger.ON_IDENTITY_AMBIGUOUS)
async def handle_ambiguous(event, ctx):
    # Multiple identities match this phone number
    # Access candidates from context, decide how to handle
    candidates = ctx.identity_result.candidates
    # Block for manual resolution, or auto-select first match
    return HookResult.block("Ambiguous identity - manual resolution required")
```

---

## Scaling & Production Questions

### How does RoomKit handle persistence?

RoomKit uses a **pluggable store** abstraction. The default is `InMemoryStore` (for development). For production, implement the `ConversationStore` interface:

```python
from roomkit import ConversationStore, Room, RoomEvent

class PostgresStore(ConversationStore):
    def __init__(self, connection_string: str):
        self._db = create_pool(connection_string)

    async def create_room(self, room: Room) -> Room:
        await self._db.execute("INSERT INTO rooms ...", room.id, ...)
        return room

    async def get_room(self, room_id: str) -> Room | None:
        row = await self._db.fetchrow("SELECT * FROM rooms WHERE id = $1", room_id)
        return self._row_to_room(row) if row else None

    # ... implement all abstract methods

kit = RoomKit(store=PostgresStore(connection_string))
```

See the [Store documentation](api/store.md) for the full interface.

### Can RoomKit scale horizontally?

Yes, with the right store backend. For multi-instance deployments:

1. Use a shared store (Postgres, Redis)
2. Use a distributed pub/sub for realtime events
3. Source connections may need coordination (one instance per source)

### How do I handle provider rate limits?

Configure retry policies and rate limits when attaching channels:

```python
from roomkit import RetryPolicy, RateLimit

await kit.attach_channel(
    room_id,
    channel_id,
    retry_policy=RetryPolicy(max_retries=3, base_delay_seconds=1.0),
    rate_limit=RateLimit(max_per_second=10),
)
```

Or implement rate limiting in hooks:

```python
@kit.hook(HookTrigger.PRE_DELIVERY)
async def rate_limit(event, ctx):
    if await is_rate_limited(ctx.channel_id):
        return HookResult.block("Rate limited - try again later")
    return HookResult.allow()
```

### How do I run multiple instances (horizontal scaling)?

Implement a distributed lock manager using `RoomLockManager`:

```python
from roomkit import RoomLockManager
from contextlib import asynccontextmanager

class RedisLockManager(RoomLockManager):
    def __init__(self, redis_client):
        self._redis = redis_client

    @asynccontextmanager
    async def locked(self, room_id: str):
        lock = self._redis.lock(f"roomkit:room:{room_id}", timeout=30)
        async with lock:
            yield

# Or use PostgreSQL advisory locks
class PostgresLockManager(RoomLockManager):
    @asynccontextmanager
    async def locked(self, room_id: str):
        lock_key = hash(room_id) & 0x7FFFFFFFFFFFFFFF
        await self._db.execute("SELECT pg_advisory_lock($1)", lock_key)
        try:
            yield
        finally:
            await self._db.execute("SELECT pg_advisory_unlock($1)", lock_key)

kit = RoomKit(store=my_store, lock_manager=RedisLockManager(redis))
```

---

## Testing Questions

### How do I test with RoomKit?

RoomKit includes mock providers and an in-memory store for testing:

```python
from roomkit import RoomKit, MockAIProvider, InMemoryStore
from roomkit.channels import AIChannel

# Create test kit with in-memory store (default)
kit = RoomKit()

# Use mock providers that don't make real API calls
ai = AIChannel("ai-test", provider=MockAIProvider(responses=["Hello!", "How can I help?"]))
kit.register_channel(ai)

# Test message flow
room = await kit.create_room("test-room")
await kit.attach_channel("test-room", "ai-test")

result = await kit.process_inbound(InboundMessage(
    channel_id="user",
    sender_id="test-user",
    content=TextContent(body="Hi"),
))

assert not result.blocked
assert result.event is not None
```

### How do I test hooks?

Hooks run during normal message processing, so test them end-to-end:

```python
blocked_messages = []

@kit.hook(HookTrigger.PRE_INBOUND, name="test_blocker")
async def block_spam(event, ctx):
    if "spam" in event.content.body.lower():
        blocked_messages.append(event)
        return HookResult.block("Spam detected")
    return HookResult.allow()

# Test the hook
result = await kit.process_inbound(spam_message)
assert result.blocked
assert len(blocked_messages) == 1
```

---

## Getting Help

- **GitHub Issues**: Bug reports and feature requests
- **Discussions**: Architecture questions and patterns
- **Examples**: See `/examples` for common integration patterns
