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
`display_name`, `role`, `status`, `identity_id`, and `connected_via`.

## One record, several channels

A participant is **one record per (room, id)**. The same person reached by SMS
and then by email is one participant, not two — that is what makes a
cross-channel identity useful. So `channel_id` is their *primary* channel, not a
filter, and `connected_via` lists every channel the room has reached them
through, primary first:

```python
await kit.add_member("team", "ws:alice:team", "alice", identity_id="alice")

# The conference asks for a participant under the same id …
p = await kit.ensure_participant("team", "conference:team", "alice")

p.channel_id     # 'ws:alice:team'  — unchanged: a lookup never re-homes a record
p.connected_via  # ['ws:alice:team', 'conference:team']
```

`ensure_participant` hands the record back **as it stands**, and logs a warning
naming both channels, because nothing in the object you receive would otherwise
tell you that the channel you named is not the one it is homed on:

```
Participant alice of room team is recorded on channel 'ws:alice:team';
'conference:team' asked for them and gets that record as it stands, primary
channel included (RFC 5.5). …
```

!!! warning "Do not keep per-channel state on a shared record"
    This is the trap the warning exists for. If you drive a lifecycle from the
    record — flipping it `ACTIVE` on connect and `LEFT` on disconnect — you are
    driving it for *every* channel that shares it. A real integration hit
    exactly this: a conference and a team channel used the same participant id,
    so leaving the call wrote `LEFT` over the team-channel membership and the
    room vanished from the user's menu. `Participant.status` is the membership
    state of that one shared record, never per-channel presence. Give each
    independently-lived channel membership a distinct, preferably namespaced
    participant id, and correlate them through `identity_id`.

`add_member` is the one verb that *may* move `channel_id` — a deliberate join
through another channel is a join through that channel — and it keeps the
channel it replaced in `connected_via`, logging the move. Recording a channel is
bookkeeping, not presentation: it emits no `PARTICIPANT_UPDATED` and fires no
`ON_PARTICIPANT_UPDATED`.

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

`LEFT` and `BANNED` differ in what they permit, not only in what they record.
`ConferenceChannel.mint_access()` reads the status before it asks the SFU for
anything: `ACTIVE`, `INACTIVE` and `LEFT` are minted for — someone whose
connection dropped asks for a second credential, not a second identity — and
`BANNED` raises `ParticipantNotAdmittedError`. It has to be caught there, since
an SFU honours the credential it minted and there is no revocation in the
backend contract. `add_member` puts a banned participant back to `ACTIVE` and
undoes it; `ensure_participant` returns the record as it stands and does not.

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
