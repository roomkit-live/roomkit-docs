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

With the advisory lock manager configured, the existing pipeline is correct
across processes; the unique constraint is the defence-in-depth backstop.

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

A worker acquires one lock-pool connection while it holds a room lock. Size
`max_size` for the number of **distinct rooms a single process handles
concurrently**. If it is too small, workers queue for a lock-pool connection
before they can even take the advisory lock.

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
