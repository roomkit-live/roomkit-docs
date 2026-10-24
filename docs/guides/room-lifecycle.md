# Room Lifecycle & Timers

Every room moves through a small set of statuses over its lifetime — from `ACTIVE`, optionally to `PAUSED`, and finally to `CLOSED`. **Room timers** let you drive those transitions automatically based on inactivity, so abandoned conversations pause and expire on their own instead of lingering forever.

## Why room timers?

A room with no end condition stays `ACTIVE` indefinitely. In production that means:

- **Cost and resource leaks** — open AI sessions, voice backends, and database rows accumulate for conversations the user walked away from.
- **Stale state** — a support thread that went silent two hours ago should not be treated the same as a live one.
- **No natural cleanup hook** — without a timer-driven close, there is no moment to archive the transcript, generate a summary, or release a phone number.

Room timers give a conversation a **time-to-live**: pause it after a short idle period, close it after a longer one, and fire lifecycle hooks at each transition so your application can react.

## Room statuses

| Status | Meaning |
|--------|---------|
| `ACTIVE` | Normal operation. |
| `PAUSED` | Soft-idle: no activity for `inactive_after_seconds`. It still accepts events, and a delivered one resumes it. `paused_at` records when it paused. The default inbound router routes the sender's next message to it as to an active room (RFC §10.4), so a paused conversation keeps its correspondent; a paused room whose binding of the channel is `group` stays its channel's room. |
| `CLOSED` | The room is finished and `closed_at` is set. Reached via `closed_after_seconds` once paused, or `close_room()`. Refuses new events. |
| `ARCHIVED` | Terminal storage state, reached with `archive_room()` — never by timers. Refuses new events. |

```
            inactive_after_seconds without activity
   ACTIVE ─────────────────────────────────────────▶ PAUSED
      ▲                                                │
      └──────────── a delivered event ─────────────────┤
                                                       │ closed_after_seconds
                                                       ▼ from paused_at
                                                    CLOSED
```

A room closes by timer only once it has paused: the close timer counts from
`paused_at`. To close a room N seconds after its last activity with no pause
between, set `inactive_after_seconds=N, closed_after_seconds=0`.

## Quick start

Pass a `RoomTimers` object when you create the room:

```python
from __future__ import annotations

from roomkit import RoomKit, RoomTimers

kit = RoomKit()

room = await kit.create_room(
    room_id="support-123",
    timers=RoomTimers(
        inactive_after_seconds=300,    # pause after 5 min without activity
        closed_after_seconds=3600,     # close after 1 h paused
    ),
)
```

To set or change the timers on an existing room, use `set_room_timers()` — no `model_copy` boilerplate:

```python
await kit.set_room_timers(
    "support-123",
    RoomTimers(closed_after_seconds=7200),   # extend the close window to 2 h
)
```

| Field | Type | Default | Effect |
|-------|------|---------|--------|
| `inactive_after_seconds` | `int \| None` | `None` | Pauses an `ACTIVE` room after N seconds without activity. `None` disables. |
| `closed_after_seconds` | `int \| None` | `None` | Closes a `PAUSED` room N seconds after `paused_at`. `None` disables. |
| `last_activity_at` | `datetime \| None` | `None` | The instant of the room's last activity, `None` while it has had none. Moved by every event the pipeline stores delivered. |

!!! note "Before any activity, the clock counts from the room's creation"
    `create_room(timers=...)` and `set_room_timers()` leave `last_activity_at`
    as it is: `None` for a room that has had no activity, whose inactivity
    then counts from its `created_at`. `set_room_timers()` keeps an existing
    activity timestamp when you only change thresholds, so adjusting a window
    mid-conversation never resets the idle clock.

## How it works

### Activity

A room's **activity** is the events the pipeline stores delivered: a message,
an event sent with `send_event()`, an agent's answer, an event a hook injected.
Each moves `last_activity_at` to its commit's instant, in the commit's
transaction. What nobody receives is no activity: a record stored `BLOCKED`
(a hook refused it, or its source is muted), the framework's lifecycle records
(an attach, a mute, a timer's transition), a record committed with
`commit_event()`, and an instruction, which is never stored. A mute does not
restart a room's inactivity timer.

### When transitions apply

Every operation that reads the room's status under its lock applies the
transitions due at its instant first: an inbound message and `send_event()`, an
agent's answer, an attach, a detach or a binding change, `commit_event()`,
`update_event()`, `delete_event()`, `close_room()`, `archive_room()`,
`start_room_recording()` and a regeneration. A message that arrives after its
room fell due is refused as if the timer had run on time.

A transition takes the instant it fell due, however late it is applied: a room
idle long past both thresholds pauses at the first instant and closes at the
second, in one write (`paused_at`, `closed_at`).

### Checking timers

A room nobody writes to transitions when you check its timers. There is no
background scheduler inside RoomKit:

```python
# Evaluate one room — returns it, its due transitions applied
room = await kit.check_room_timers("support-123")

# Evaluate every ACTIVE / PAUSED room — returns the ones that transitioned
transitioned = await kit.check_all_timers()

# Multi-tenant scheduler: never sweep another organization's rooms
transitioned = await kit.check_all_timers(organization_id="acme")
```

### Driving the sweep

Run `check_all_timers()` periodically from a background task (or an external cron / scheduler):

```python
import asyncio
import logging

logger = logging.getLogger("app.timers")


async def timer_sweep(kit: RoomKit, interval: float = 60.0) -> None:
    while True:
        await asyncio.sleep(interval)
        for room in await kit.check_all_timers():
            logger.info("Room %s transitioned to %s", room.id, room.status)


sweep_task = asyncio.create_task(timer_sweep(kit))
```

A late sweep closes a room at the instant it fell due, not at the sweep's, so
the interval only decides how soon a room nobody writes to is seen closed.

## Lifecycle hooks

Each timer transition emits its framework event, `room_paused` or `room_closed`
with `reason = timer`, as soon as it is stored and before anything the
operation decides next: a message a due close refuses reports `room_closed`
before its `room_refused_event`. Its hook fires once the room lock is released
(`check_room_timers()` waits for it before returning), with the room as the
transition left it:

```python
from roomkit import HookExecution, HookTrigger

@kit.hook(HookTrigger.ON_ROOM_PAUSED, execution=HookExecution.ASYNC)
async def on_paused(event, ctx):
    logger.info("Room %s paused at %s", ctx.room.id, ctx.room.paused_at)

@kit.hook(HookTrigger.ON_ROOM_CLOSED, execution=HookExecution.ASYNC)
async def on_closed(event, ctx):
    # archive conversation, generate summary, free the phone number…
    ...
```

The transition also writes a lifecycle record into the room —
`room_paused_by_timer` or `room_closed_by_timer`, `internal`, never delivered —
whose `data` carries the `threshold` that elapsed and the instant (`at`) it
fell due, useful for auditing why a room closed. The hook's event carries the
same.

## Resuming a paused room

`PAUSED` is a soft state, not a dead end: an event the pipeline stores
delivered into a paused room resumes it, in the same transaction — `ACTIVE`
again, `paused_at` cleared, its idle clock restarted. A resume fires no hook
and emits no framework event. What is no activity resumes nothing, a provider's
delivery status included.

The sender's next message finds their paused room: routing reads ACTIVE and
PAUSED rooms alike when it looks for the sender's own room, and the newest
wins whatever its status. A room whose close timer fell due is closed before
the router's choice is checked under the room lock, so the message then gets a
new room rather than a `room_closed` refusal.

To resume a paused room without a message, set its status back to `ACTIVE`
and restart its idle clock:

```python
from datetime import UTC, datetime

from roomkit.models import RoomStatus

# Under the room's lock: a whole-room write from a room read earlier would
# undo what a message committed meanwhile wrote (its counters, the room's
# register of authors).
async with kit.lock_manager.locked("support-123"):
    room = await kit.get_room("support-123")
    timers = room.timers.model_copy(update={"last_activity_at": datetime.now(UTC)})
    await kit.store.update_room(
        room.model_copy(update={"status": RoomStatus.ACTIVE, "paused_at": None, "timers": timers})
    )
```

## Manual close

Timers are optional. You can always end a room directly, which sets `closed_at` and fires `ON_ROOM_CLOSED`:

```python
await kit.close_room("support-123")
```

A transition the room has already made does nothing: closing a `CLOSED` room
returns it as stored, writing nothing and firing nothing. No call takes a room
out of `ARCHIVED`: closing an archived room raises `TransitionRefusedError`
(its `room_id`, `status` and `transition`), nothing written.

## Archiving

`archive_room()` moves a room to `ARCHIVED`, the terminal storage state:

```python
await kit.archive_room("support-123")
```

It refuses new events exactly as closing does — the difference is intent.
A closed room may be reopened by returning it to `ACTIVE`; an archived one is
done. Reading is unaffected at every status: history, participants and bindings
stay readable in an archived room.

`regenerate_response()` is refused the same way at either status, before the
agent runs: it returns `InboundResult(blocked=True, reason="room_closed")`, and
`regenerate_target()` raises `RoomClosedError`, so a host learns the room is
closed before it acts on a trigger.

Archiving an `ARCHIVED` room returns it as stored, writing and emitting
nothing; otherwise the call stops any room-level media recorder and emits the
`room_archived` framework event:

```python
@kit.on("room_archived")
async def on_archived(event):
    await warehouse.export(event.room_id)
```

## Example

See [`examples/room_lifecycle.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/room_lifecycle.py) for a runnable demo covering manual close, timer-driven auto-pause / auto-close, and batch checking with `check_all_timers()`.

## Related guides

- [Production Resilience](production-resilience.md) — circuit breakers, retries, rate limiting, and where room timers fit in a full production setup.
- [Activity Persistence](activity-persistence.md) — persisting rooms and events so timers survive restarts.
- [Telemetry](telemetry.md) — observing lifecycle transitions in production.
