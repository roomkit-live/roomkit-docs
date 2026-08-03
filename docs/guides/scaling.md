# Horizontal Scaling (Multi-Process)

RoomKit runs happily as a single process out of the box. Once you put several
RoomKit processes behind a load balancer — all sharing **one** PostgreSQL store —
you must add cross-process coordination, or concurrent workers can corrupt a
room's event ordering. This guide covers what changes and how to configure it.

## Why a shared store needs more than the default lock

Every event in a room gets a sequential `index` (0, 1, 2, …) that is unique and
monotonically increasing *per room* (RFC §8.1). Pagination cursors
(`after_index` / `before_index`), read markers ("seen by"), timeline ordering,
and threading all depend on that invariant.

The default `InMemoryLockManager` serializes event processing **within one
process**. It cannot coordinate across processes — each worker holds its own
in-memory lock. So with two workers sharing one database:

```text
                Worker A                     Worker B
  t1   get_event_count(room) → 5
  t2                                get_event_count(room) → 5
  t3   assign index = 5, INSERT
  t4                                assign index = 5, INSERT   ← same index!
```

Both events land at `index = 5`. Nothing serializes them, and — without a unique
constraint — the duplicate is stored silently, quietly breaking every reader that
relies on the index.

This only appears under horizontal scaling, so it is easy to ship a single-process
app that corrupts data the moment it is scaled out.

## The two safeguards

RoomKit closes this with a coordinator plus a database backstop (RFC §13.5):

1. **`PostgresAdvisoryLockManager`** — a `RoomLockManager` that serializes room
   processing *across processes* using PostgreSQL session advisory locks.
2. **`UNIQUE(room_id, index)`** on the `events` table — a duplicate index is
   rejected loudly (a constraint violation) instead of silently persisted. The
   `PostgresStore` schema applies it automatically.

With the advisory lock manager configured, the pipeline is correct across
processes; the unique constraint is the defence-in-depth backstop.

Note what the room lock does — and deliberately does not — cover. It
serializes everything that decides and writes the timeline: the pre-commit
gates, index assignment, the atomic commit, and the *planning* of the
broadcast (RFC §10.1 steps 6–12). **External delivery executes off the
lock**, in per-room delivery lanes ordered by the store's `delivered_index`
cursor — see [Delivery lanes](#delivery-lanes-ordered-delivery-off-the-room-lock)
below. A slow provider or a long AI generation therefore no longer extends
the room's critical section, which under a distributed lock manager is what
serializes the whole deployment.

## Delivery lanes: ordered delivery off the room lock

Every committed event's delivery set (which channels get `on_event` /
`deliver`) is resolved under the room lock, then executed by the room's
*delivery lane* — strictly in index order, one event's set completing before
the next begins (RFC §10.2). Order across workers comes from two shared
pieces, not from lock tenure:

- **`Room.delivered_index`** — a per-room cursor the store advances by strict
  compare-and-set: a lane may only execute the plan at `delivered_index + 1`.
- **The delivery claim** — a derived `__delivery__:{room_id}` key on the lock
  manager, held while a lane executes. It serializes the executors of one
  room across processes, and it is why a *slow* delivery is never mistaken
  for a dead worker: while the owner works, the claim is held and the
  waiters block on acquisition instead of measuring a gap.

A worker that commits events and **crashes before delivering them** releases
its claim with its connection and leaves a cursor hole. The next lane with
work for that room waits `delivery_gap_timeout` (default 30 s,
`RoomKit(delivery_gap_timeout=...)`), then skips over the hole and emits a
**`delivery_skipped`** framework event (`{from_index, to_index}`). The loss
is bounded to exactly what the crashed worker had committed-but-undelivered —
the same window as a crash under the previous under-lock delivery — but it is
now observable, and the room never wedges. Subscribe to `delivery_skipped`
if you want to alert on it or re-send from your own records.

Callers still observe their own delivery: `process_inbound` / `send_event`
return once their event's delivery set *and* every AI reentry pass it
spawned have executed. The one relaxation (RFC §10.1 step 14): a concurrent
inbound may commit between a trigger and its response — ordering guarantees
are per-room index monotonicity and parent linkage, never adjacency.

### One delivery primitive per room

The cursor advances **after** an event's delivery set has run, never at
commit time — advancing it early would declare the event delivered and
release the lane to execute the next index while this one is still on its
way out. That is why every committed event with recipients goes through the
lane, including the ones a caller produces itself: a greeting, a regenerated
answer, the segments of a streamed reply. A streamed answer's segments
therefore reach the non-streaming channels *as they are produced* rather
than in a batch after the stream, each firing its `AFTER_BROADCAST` once its
own delivery set completes — an SMS participant follows the answer at the
pace the web one does.

The flip side is head-of-line blocking, and it is the specified behaviour
(RFC §10.2): while a room is streaming a long answer, the delivery of events
committed *after* those segments waits for them. It is per room only —
other rooms' lanes are untouched — and it is the price of "one event's set
completes before the next begins". If a room needs a side channel that does
not queue behind a turn, give it its own room.

### The claim pool

A claim is held for the length of a delivery set — provider round trips and
AI generation included — while a room lock is now held only for gates,
commit and planning. On a shared `PostgresAdvisoryLockManager`, long claim
tenures and short commit tenures would compete for the same connections, so
give the claims their own manager in multi-process deployments:

```python
claims = PostgresAdvisoryLockManager(
    dsn="postgresql://user:pass@db/roomkit",
    max_size=20,   # ≈ rooms this worker delivers for concurrently
)
await claims.init()

kit = RoomKit(
    store=store,
    lock_manager=locks,
    delivery_claim_lock_manager=claims,
)
```

The default (claims on the room-lock manager) is fine for a single process —
`InMemoryLockManager` has no pool to starve.

## Configuration

Pair a `PostgresStore` with a `PostgresAdvisoryLockManager`:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.store.postgres import PostgresStore
from roomkit.store.postgres_lock import PostgresAdvisoryLockManager

store = PostgresStore(dsn="postgresql://user:pass@db/roomkit")
await store.init()

# IMPORTANT: give the lock manager its OWN connection pool (a separate DSN, or
# just separate credentials/pool), NOT the store's. A session advisory lock is
# held on a connection for the whole locked section; sharing the store's query
# pool could let held lock connections starve the queries that the locked
# section needs, deadlocking.
locks = PostgresAdvisoryLockManager(
    dsn="postgresql://user:pass@db/roomkit",
    max_size=20,   # ≈ number of rooms processed concurrently
)
await locks.init()

kit = RoomKit(store=store, lock_manager=locks)
# ... use kit ...
await kit.close()   # closes the store and the lock manager pools
```

### Pool sizing

A worker acquires one lock-pool connection while it holds a room lock. Since
delivery moved off the lock, that tenure is short — gates, commit, broadcast
planning; no provider I/O — so `max_size` sizes for the number of **distinct
rooms a single process commits for concurrently**. If it is too small,
workers queue for a lock-pool connection before they can even take the
advisory lock. The *long* tenures live on the delivery claims: size the
claim manager's pool (see [The claim pool](#the-claim-pool)) for the rooms a
worker delivers for concurrently.

### The startup warning

If you point RoomKit at a persistent store while keeping the default in-memory
lock, it warns at construction:

```text
PostgresStore is paired with InMemoryLockManager. This is safe only in a single
process; if the store is shared across processes (e.g. a load-balanced
deployment), use a distributed lock manager such as PostgresAdvisoryLockManager
to avoid duplicate event indices.
```

Single-process deployments can ignore it; multi-process deployments must act on
it.

## Migrating an existing database

`PostgresStore.init()` applies `UNIQUE(room_id, index)` automatically:

- **Fresh or already-clean database** → the unique index is created. Nothing to do.
- **Database that already contains duplicate indices** (from running a pre-fix
  release under concurrency) → `init()` **cannot** create the unique index. It
  does **not** crash: it keeps the existing non-unique index, logs a warning, and
  starts. Multi-process safety is not enforced until you deduplicate.

```text
idx_events_room_index is not UNIQUE — duplicate (room_id, index) rows exist, so
multi-process event-index safety is NOT enforced. Deduplicate the events table,
then recreate the index UNIQUE.
```

### Repairing duplicates

Use the built-in, transactional repair. It renumbers each affected room's events
to a unique, sequential `0..N-1` (ordered by `index`, then `created_at`, then
`id`), reconciles the room counters, and (re)creates the unique index.

```python
store = PostgresStore(dsn="postgresql://user:pass@db/roomkit")
await store.init()

# 1) Dry run — reports what would change, touches nothing:
print(await store.dedupe_event_indices())
# → {"action": "dry_run", "duplicate_rows": 12, "affected_rooms": 3, "now_unique": False}

# 2) Apply the repair (renumber + enforce UNIQUE, one transaction):
print(await store.dedupe_event_indices(dry_run=False))
# → {"action": "repaired", "duplicate_rows": 12, "affected_rooms": 3, "now_unique": True}
```

!!! warning "Read markers shift"
    Renumbering changes event indices, so read markers (`last_read_index` /
    `read_markers.event_index`) can be off once afterwards (a stray "seen by").
    Run the repair in a maintenance window, on a backup first, and prefer to do
    it **before** scaling out (so no concurrent workers race during the repair).

## Distributed ephemeral surfaces

The store and locks above cover persistent state. Two ephemeral surfaces are
process-local by default and stay silent about it — a second worker simply
never sees the other's events:

- **Realtime events** (typing, presence, reactions, thinking deltas):
  `InMemoryRealtime` only reaches subscribers in the same process. Use
  [`RedisRealtimeBackend`](realtime-features.md#distributed-deployments-redisrealtimebackend)
  so ephemeral events cross workers via Redis pub/sub.
- **Status bus** (multi-agent coordination): `InMemoryStatusBackend` keeps
  history and subscribers per-process. Use
  [`RedisStatusBackend`](status-bus.md#pluggable-backends) for a shared
  capped history plus cross-process notifications.

```python
from roomkit import RoomKit
from roomkit.orchestration import RedisStatusBackend
from roomkit.orchestration.status_bus import StatusBus
from roomkit.realtime import RedisRealtimeBackend

kit = RoomKit(
    store=store,
    lock_manager=lock_manager,
    realtime=RedisRealtimeBackend("redis://redis:6379"),
    status_bus=StatusBus(backend=RedisStatusBackend("redis://redis:6379")),
)
```

Both require `pip install roomkit[redis]`. For persistent cross-worker
*delivery* (queued outbound messages surviving restarts), see the separate
[`RedisDeliveryBackend`](delivery.md#redis-backend).

## Checklist before scaling out

1. Apply `UNIQUE(room_id, index)` on a **clean** database (run `init()` or
   `dedupe_event_indices(dry_run=False)`) — do this before adding a second worker.
2. Configure `PostgresAdvisoryLockManager` with its own pool on every worker.
3. Confirm no startup warning about `InMemoryLockManager`.
4. Point all workers at the same PostgreSQL store.
5. If clients rely on typing/presence or agents on the status bus, configure
   the Redis realtime and status backends on every worker.

## Scaling voice inside one worker

Horizontal scaling addresses room count; a voice worker's ceiling is
per-process CPU. Set `AudioPipelineConfig(inbound_dsp_threads=N)` to run
each session's DSP chain on a thread pool instead of the event loop — see
[Audio Pipeline Stages](audio-pipeline-stages.md#running-the-inbound-chain-off-the-event-loop).

## See also

- [PostgreSQL Storage](postgres-store.md) — the store, schema, and `migrate()`.
- RFC [§8.1 (event indexing)](https://github.com/roomkit-live/roomkit-specs/blob/main/roomkit-rfc.md)
  and §13.5 (room-level locking) — the normative invariants behind this guide.
