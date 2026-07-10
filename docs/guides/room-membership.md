# Room Membership

RoomKit distinguishes **explicit membership** — a deliberate join or leave —
from the lazy participant that [identity resolution](identity-resolution.md)
materialises the first time someone speaks. `add_member` / `remove_member` model
an intentional roster (invite a user, remove one), while `ensure_participant`
keeps handling the "sender we've never seen before" case underneath.

Membership is a soft, auditable state: leaving flips a status rather than
deleting a row, so history and read markers survive, and every transition emits
a system event and fires a lifecycle hook.

## Quick start

```python
from roomkit import RoomKit, WebSocketChannel

kit = RoomKit()
kit.register_channel(WebSocketChannel("ws-alice"))
await kit.create_room(room_id="team")
await kit.attach_channel("team", "ws-alice")

# A deliberate join
await kit.add_member(
    "team",
    channel_id="ws-alice",
    participant_id="alice",
    identity_id="alice",        # marks the participant IDENTIFIED
    display_name="Alice",
)
```

## Joining

`add_member` is **idempotent** and safe to call on every room open: joining
someone who is already an `ACTIVE` member is a no-op — no write, no event, no
hook. A genuine join (a brand-new member, or re-adding someone who previously
left) upserts the participant as `ACTIVE`, emits `PARTICIPANT_JOINED`, and fires
the `ON_PARTICIPANT_JOINED` hook. A re-join preserves the original `joined_at`.

```python
await kit.add_member("team", channel_id="ws-alice", participant_id="alice")  # no-op if active
```

When `identity_id` is supplied the participant is marked `IDENTIFIED` (you
already know who they are); otherwise it stays `PENDING` for the identity
pipeline to resolve. `role` defaults to `ParticipantRole.MEMBER` — pass
`OWNER`, `AGENT`, `OBSERVER`, or `BOT` as needed.

!!! note "`add_member` vs `ensure_participant`"
    `add_member` is an **intentional** join you drive from your application.
    `ensure_participant` is the framework's **lazy** materialisation of a sender
    inside the inbound pipeline. They share the participant store and are
    idempotent by `participant_id`, so the two coexist without duplicates.

## Reading the roster

```python
active = await kit.list_members("team")                    # ACTIVE only (the current roster)
everyone = await kit.list_members("team", include_left=True)  # includes LEFT / BANNED
joined = await kit.is_member("team", "alice")              # True if that identity is ACTIVE
```

`list_members` returns `Participant` objects — each carries `channel_id`,
`display_name`, `role`, `status`, and `identity_id`.

## Leaving

`remove_member` is a **soft** leave: it flips `status` to `LEFT` (or pass
`BANNED`) rather than deleting the row, so membership history and read markers
survive. It emits `PARTICIPANT_LEFT`, fires the `ON_PARTICIPANT_LEFT` hook, and
raises `ParticipantNotFoundError` if the participant is unknown.

```python
from roomkit import ParticipantNotFoundError
from roomkit.models.enums import ParticipantStatus

await kit.remove_member("team", "carol")                              # status -> LEFT
await kit.remove_member("team", "mallory", status=ParticipantStatus.BANNED)  # status -> BANNED
```

## Lifecycle hooks

Each transition fires an async lifecycle hook, so you can run side effects —
announce the join, provision resources, archive on leave:

```python
from roomkit import HookExecution, HookTrigger

@kit.hook(HookTrigger.ON_PARTICIPANT_JOINED, execution=HookExecution.ASYNC)
async def on_joined(event, ctx):
    print("joined:", event.content.data["participant_id"])

@kit.hook(HookTrigger.ON_PARTICIPANT_LEFT, execution=HookExecution.ASYNC)
async def on_left(event, ctx):
    print("left:", event.content.data["participant_id"], event.content.data["status"])
```

The matching `PARTICIPANT_JOINED` / `PARTICIPANT_LEFT` system events are also
written to the room timeline for auditing.

## "Seen by" read markers

`read_markers` is the **single source of truth** for read position. Each channel
advances its marker with [`mark_read` / `mark_all_read`](../api/store.md); with
one channel per member, aggregating the markers gives a per-member "seen by"
receipt. `ChannelBinding.last_read_index` is an explicitly *non-authoritative*
hint that the read API does not advance — do not read receipts from it.

`list_read_markers` returns every channel's read high-water-mark as
`channel_id -> event index`. Map the channel back to a member via the roster:

```python
markers = await kit.list_read_markers("team")   # {"ws-bob": 8, "ws-carol": 6}
by_channel = {p.channel_id: p.display_name for p in await kit.list_members("team")}

for channel_id, index in markers.items():
    print(f"{by_channel.get(channel_id, channel_id)} has read up to event {index}")
```

Channels with no marker are simply absent from the mapping. To turn a marker
into an unread badge, compare it against the room's latest event index (or use
`kit.store.get_unread_count(room_id, channel_id)` for a single channel).

## Example

See [`examples/room_membership.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/room_membership.py)
for a complete, runnable walkthrough — join, roster reads, hooks, "seen by"
aggregation, and a soft leave.

## Related guides

- [Room Lifecycle & Timers](room-lifecycle.md) — room-level status (`ACTIVE` /
  `PAUSED` / `CLOSED`) and idle-driven transitions, the room-scoped counterpart
  to member-scoped join/leave.
- [Identity Resolution](identity-resolution.md) — how anonymous senders become
  identified participants that `add_member` can pin to an identity.
- [Message Threading](message-threading.md) — replies and thread roots on the
  same event model.
