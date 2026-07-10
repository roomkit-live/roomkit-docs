# Message Threading

RoomKit supports **flat, two-level threads** (Slack / Teams style): a **root**
message and its **replies**. A reply carries `parent_event_id` pointing at its
thread root; a root or non-threaded message has `parent_event_id = None`.

Threading is transport-agnostic — the parent is applied centrally in the
inbound pipeline, so it works for WebSocket, SMS, email, and any other channel
without per-channel wiring. It is distinct from
[`ChannelData.thread_id`](#relationship-to-channeldatathread_id), which is a
provider-native thread reference.

## Quick start

Set `parent_event_id` on the inbound message (or on `send_event`) to reply
inside a thread:

```python
from roomkit import InboundMessage, RoomKit, TextContent

kit = RoomKit()
# ... register/attach channels, create room "r1" ...

root = await kit.process_inbound(
    InboundMessage(channel_id="ws-alice", sender_id="alice",
                   content=TextContent(body="Can we move the sync?"))
)
root_id = root.event.id

# A reply in that thread
await kit.process_inbound(
    InboundMessage(channel_id="ws-bob", sender_id="bob",
                   content=TextContent(body="Sure — which day?"),
                   parent_event_id=root_id)
)

# Or via direct injection
await kit.send_event("r1", "ws-bob", TextContent(body="Tuesday?"), parent_event_id=root_id)
```

## The flat two-level invariant

`parent_event_id` **always points at a thread root**. If you reply to a message
that is *itself* a reply, RoomKit normalises the pointer to that reply's root —
so a thread never nests beyond one level:

```python
reply = ...        # parent_event_id == root_id
nested = await kit.send_event("r1", "ws-alice", TextContent(body="Tuesday?"),
                              parent_event_id=reply.id)
assert nested.parent_event_id == root_id   # collapsed to the root, not reply.id
```

Normalisation happens once, inside the locked pipeline, for every entry point
(`process_inbound` and `send_event` alike). A `parent_event_id` that does not
exist or belongs to another room drops to top level (`None`) with a warning —
a stale reference never loses the sender's message.

## AI replies stay in the thread

When an intelligence channel is triggered by a threaded message, its response
**inherits the trigger's thread root** — so an `@`-mention inside a thread is
answered inside that thread. A mention at top level is answered at top level.
This holds on both the streaming and non-streaming generation paths, and no
configuration is required.

## Reading threads

Two [`EventFilter`](../api/store.md) fields drive thread-aware reads:

```python
from roomkit.models.store_filter import EventFilter

# Main timeline: roots + standalone messages, replies excluded
timeline = await kit.store.list_events("r1", event_filter=EventFilter(top_level_only=True))

# One thread: the replies of a given root
thread = await kit.store.list_events("r1", event_filter=EventFilter(parent_event_id=root_id))
```

`top_level_only` and `parent_event_id` are mutually exclusive.

### Reply counts

`get_thread_summaries` returns per-root aggregates so you can render a
"N replies · last reply" affordance without fetching every reply:

```python
summaries = await kit.store.get_thread_summaries("r1", [root_id])
summary = summaries[root_id]         # roots with no replies are absent
summary.reply_count                  # e.g. 4
summary.last_reply_at                # datetime of the latest reply
```

## Capability

Channels that support threading advertise it via
`ChannelCapabilities.supports_threading` (true for the in-app WebSocket channel,
Email, Teams, and Discord). The value is informational — it tells the AI what
the target channel supports; it does not gate the storage behaviour above.

## Storage

Replies are ordinary events with `parent_event_id` set — no separate table.
The PostgreSQL store keeps a partial index on `events(parent_event_id)` for
efficient thread reads. Because a thread is fixed at creation, `parent_event_id`
is set on insert and never mutated.

## Relationship to `ChannelData.thread_id`

`parent_event_id` is RoomKit's **in-app** threading key. `ChannelData.thread_id`
is a **provider-native** reference (Slack `thread_ts`, Discord message id, Teams
`replyToId`) passed straight through to/from the provider. They are independent;
bridging a provider's native threads to in-app threads is a separate concern.

## Example

See [`examples/message_threading.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/message_threading.py)
for a complete, runnable walkthrough.
