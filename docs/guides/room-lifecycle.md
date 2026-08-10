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
| `ACTIVE` | Normal operation. The default inbound router only routes messages to active rooms. |
| `PAUSED` | Soft-idle. The room stopped receiving new traffic but is not yet closed. Reached via `inactive_after_seconds`. |
| `CLOSED` | Terminal. The room is finished and `closed_at` is set. Reached via `closed_after_seconds` or `close_room()`. |
| `ARCHIVED` | Terminal storage state, reached with `archive_room()` — never by timers. |

```
                inactive_after_seconds idle
   ACTIVE ───────────────────────────────────▶ PAUSED
      │                                            │
      │ closed_after_seconds idle                  │ closed_after_seconds idle
      ▼                                            ▼
   CLOSED ◀──────────────────────────────────────
```

The close timer **supersedes** the pause timer: once the close threshold is crossed, the room goes straight to `CLOSED` whether it was `ACTIVE` or already `PAUSED`.

## Quick start

Pass a `RoomTimers` object when you create the room:

```python
from __future__ import annotations

from roomkit import RoomKit, RoomTimers

kit = RoomKit()

room = await kit.create_room(
    room_id="support-123",
    timers=RoomTimers(
        inactive_after_seconds=300,    # pause after 5 min idle
        closed_after_seconds=3600,     # close after 1 h idle
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
| `inactive_after_seconds` | `int \| None` | `None` | Pauses an `ACTIVE` room after N seconds of inactivity. `None` disables. |
| `closed_after_seconds` | `int \| None` | `None` | Closes the room after N seconds of inactivity. Takes priority over the pause timer. `None` disables. |
| `last_activity_at` | `datetime \| None` | `None` | Timestamp the idle clock measures from. Updated automatically on every processed inbound event. |

!!! note "The idle clock starts automatically"
    Both `create_room(timers=...)` and `set_room_timers()` fill in `last_activity_at` for you when you omit it, so the countdown begins immediately. `set_room_timers()` also **preserves** an existing activity timestamp when you only change thresholds, so adjusting a window mid-conversation never resets the idle clock. Timers stay inert only if you build a `Room` by hand and leave `last_activity_at` as `None`.

## How it works

### Activity tracking

Every inbound message that is processed for a room updates `last_activity_at` to the current time. An active back-and-forth keeps resetting the idle clock, so timers only fire once the conversation actually goes quiet — you do not need to bump the timestamp yourself.

### Checking timers

There is **no background scheduler inside RoomKit**. Timer thresholds are only evaluated when you ask for them:

```python
# Evaluate one room — returns it, possibly transitioned
room = await kit.check_room_timers("support-123")

# Evaluate every ACTIVE / PAUSED room — returns the ones that transitioned
transitioned = await kit.check_all_timers()

# Multi-tenant scheduler: never sweep another organization's rooms
transitioned = await kit.check_all_timers(organization_id="acme")
```

`check_room_timers()` computes `elapsed = now - last_activity_at`, applies the close threshold first, then the pause threshold, and persists any status change. `check_all_timers()` sweeps all active and paused rooms and returns just the list that changed.

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

Pick an interval shorter than your tightest threshold — a 60-second sweep is fine for minute-scale idle timeouts.

## Lifecycle hooks

Each timer transition fires a lifecycle hook so you can run side effects — notify participants, archive the transcript, generate a summary, release resources:

```python
from roomkit import HookExecution, HookTrigger

@kit.hook(HookTrigger.ON_ROOM_PAUSED, execution=HookExecution.ASYNC)
async def on_paused(event, ctx):
    logger.info("Room %s paused due to inactivity", ctx.room.id)

@kit.hook(HookTrigger.ON_ROOM_CLOSED, execution=HookExecution.ASYNC)
async def on_closed(event, ctx):
    # archive conversation, generate summary, free the phone number…
    ...
```

Alongside the hooks, the transition emits a system event into the room — `room_paused_by_timer` or `room_closed_by_timer` — carrying `elapsed_seconds` and the `threshold` that fired, useful for auditing why a room closed.

## Resuming a paused room

`PAUSED` is a soft state, not a dead end — but RoomKit does **not** auto-resume it. The default inbound router only routes to `ACTIVE` rooms, so a paused room receives no new traffic and its idle clock keeps running toward the close threshold. If your application decides a paused conversation should continue, set its status back to `ACTIVE`:

```python
from roomkit.models.enums import RoomStatus

room = await kit.get_room("support-123")
room = room.model_copy(update={"status": RoomStatus.ACTIVE})
await kit.store.update_room(room)
```

Once active again, inbound traffic resumes and each new message refreshes `last_activity_at`. To restart the idle clock immediately without waiting for a message, call `set_room_timers()` with a fresh `last_activity_at`.

## Manual close

Timers are optional. You can always end a room directly, which sets `closed_at` and fires `ON_ROOM_CLOSED`:

```python
await kit.close_room("support-123")
```

## Archiving

`archive_room()` moves a room to `ARCHIVED`, the terminal storage state:

```python
await kit.archive_room("support-123")
```

It refuses new events exactly as closing does — the difference is intent.
A closed room may be reopened by returning it to `ACTIVE`; an archived one is
done. Reading is unaffected at every status: history, participants and bindings
stay readable in an archived room.

The call is idempotent, stops any room-level media recorder, and emits the
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
