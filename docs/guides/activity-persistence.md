# Activity Persistence

When an AI agent uses tools during a conversation, the full timeline matters: which tools were called, with what arguments, what they returned, and how long they took. RoomKit persists this interleaving as individual events rather than flattening everything into a single text blob.

## The Problem

Without activity persistence, a streaming AI response like:

```
text: "Let me search for that..."
tool: web_search(q="roomkit docs") -> {results: [...]}
text: "I found 3 results. Let me fetch the page..."
tool: web_fetch(url="...") -> {html: "..."}
text: "Here's what I found: ..."
```

Would be stored as a single concatenated string, losing tool call boundaries, arguments, results, and timestamps. On reload, the client sees one wall of text with no trace of what the AI actually did.

## How It Works

RoomKit persists each segment of an AI response as a separate `RoomEvent`:

| Index | Type | Content |
|-------|------|---------|
| 1 | `message` | "Let me search for that..." |
| 2 | `tool_call_start` | `web_search`, args: `{q: "roomkit docs"}`, status: pending |
| 3 | `tool_call_end` | `web_search`, result: `{results: [...]}`, status: completed, 1200ms |
| 4 | `message` | "I found 3 results. Let me fetch..." |
| 5 | `tool_call_start` | `web_fetch`, args: `{url: "..."}`, status: pending |
| 6 | `tool_call_end` | `web_fetch`, result: `{html: "..."}`, status: completed, 800ms |
| 7 | `message` | "Here's what I found: ..." |

All events share the same `correlation_id`, linking them as parts of one AI response. Each has its own `created_at` timestamp and sequential `index`.

This works identically for both streaming and non-streaming AI providers.

## ToolCallContent

Tool call events use the `ToolCallContent` content type:

```python
from roomkit import ToolCallContent

# At tool start (persisted automatically by the framework)
ToolCallContent(
    tool_name="web_search",
    tool_id="call-abc123",
    arguments={"q": "roomkit docs"},
    status="pending",
)

# At tool end
ToolCallContent(
    tool_name="web_search",
    tool_id="call-abc123",
    arguments={"q": "roomkit docs"},
    result={"results": [...]},
    status="completed",  # or "failed"
    duration_ms=1200,
    error=None,  # populated on failure
)
```

## Querying the Store

### Full Timeline

Retrieve everything that happened in a room, in order:

```python
timeline = await store.get_timeline(room_id)

for event in timeline:
    if event.type == EventType.MESSAGE:
        print(f"[{event.created_at}] Text: {event.content.body}")
    elif event.type == EventType.TOOL_CALL_START:
        print(f"[{event.created_at}] Tool started: {event.content.tool_name}")
    elif event.type == EventType.TOOL_CALL_END:
        print(f"[{event.created_at}] Tool done: {event.content.tool_name} ({event.content.duration_ms}ms)")
```

### Conversation Only (AI Context)

For rebuilding AI context, use `get_conversation()` which returns only `MESSAGE` events -- no tool call noise:

```python
messages = await store.get_conversation(room_id, limit=50)
# Only text messages, suitable for feeding back to the AI provider
```

Without a cursor this returns the **most recent** `limit` messages, in
ascending order -- `messages[-1]` is the newest message in the room. This is
what fills `RoomContext.recent_events`, so a room whose history outgrew the
limit still hands hooks and AI channels the current conversation, not its
opening.

Pass `after_index` to turn it into a forward cursor instead -- the first
`limit` messages *after* that index, which is what you want to page through
everything that arrived since your last read:

```python
new_messages = await store.get_conversation(room_id, limit=50, after_index=last_seen)
```

### Filtering with EventFilter

`EventFilter` provides fine-grained control over queries:

```python
from roomkit import EventFilter, EventType

# Get all tool calls in a room
tools = await store.list_events(
    room_id,
    event_filter=EventFilter(
        event_types=[EventType.TOOL_CALL_START, EventType.TOOL_CALL_END],
    ),
)

# Get events from a specific AI response (by correlation_id)
response_events = await store.list_events(
    room_id,
    event_filter=EventFilter(correlation_id="abc123"),
)

# Get events from a specific channel
ai_events = await store.list_events(
    room_id,
    event_filter=EventFilter(source_channel_id="ai-assistant"),
)

# Time-range queries
from datetime import datetime, UTC, timedelta

recent = await store.list_events(
    room_id,
    event_filter=EventFilter(
        after_time=datetime.now(UTC) - timedelta(hours=1),
    ),
)

# Combine filters
recent_tools = await store.list_events(
    room_id,
    event_filter=EventFilter(
        event_types=[EventType.TOOL_CALL_END],
        source_channel_id="ai-assistant",
        after_time=datetime.now(UTC) - timedelta(hours=1),
    ),
)
```

## Persistence Policy

Control which event types are persisted with `PersistencePolicy`:

```python
from roomkit import RoomKit, PersistencePolicy, EventType

# Persist everything except typing and presence indicators
kit = RoomKit(
    persistence_policy=PersistencePolicy(
        exclude_types={EventType.TYPING, EventType.PRESENCE},
    ),
)

# Or whitelist specific types
kit = RoomKit(
    persistence_policy=PersistencePolicy(
        persist_types={
            EventType.MESSAGE,
            EventType.TOOL_CALL_START,
            EventType.TOOL_CALL_END,
        },
    ),
)
```

`exclude_types` always takes precedence over `persist_types`. When `persist_types` is `None` (default), all event types are persisted.

The policy is checked before every `add_event` call in the framework. Events excluded by the policy are still broadcast to channels in real-time -- they just aren't written to the store.

## Architecture

### Streaming Path

During streaming, the AI generator yields structured markers alongside text deltas:

```
str("Let me search...")  ->  transport channels (real-time display)
ToolCallStartMarker      ->  triggers text segment persistence
ToolCallEndMarker        ->  triggers tool call end persistence
str("Here are results")  ->  transport channels (real-time display)
```

Transport channels only see string deltas for real-time rendering. The framework's streaming consumer intercepts markers and persists events at each boundary.

### Non-Streaming Path

The non-streaming tool loop tracks each round (text + tool calls + results + duration) and builds the interleaved event list when the response is complete. All events are returned via `ChannelOutput.response_events` and persisted by the inbound pipeline.

### Broadcast Behavior

Tool call events (`TOOL_CALL_START`, `TOOL_CALL_END`) are **persisted but not broadcast** through the event router. This prevents intelligence channels (other AI agents) from trying to respond to tool call events. Text segments are broadcast normally.

`AFTER_BROADCAST` hooks fire for all event types, so hook authors can observe tool calls for analytics, logging, or auditing.

## Related

- [Streaming with Tools](streaming-tools.md) -- real-time streaming delivery of tool-using AI responses
- [Tool Calling](tool-calling.md) -- tool definition, policies, and execution
- [Auditing](tool-audit.md) -- session and tool audit trails
- [PostgreSQL Storage](postgres-store.md) -- production storage backend with EventFilter SQL support
