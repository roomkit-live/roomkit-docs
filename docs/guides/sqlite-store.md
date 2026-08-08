# SQLite Storage

`SQLiteStore` is RoomKit's embedded persistent `ConversationStore`. It stores a
complete conversation in one SQLite file, needs no server or optional Python
dependency, and keeps disk I/O off the event loop with a dedicated worker
thread.

Use it for a desktop application, an edge device, or a small bot that runs as
one process. Use `PostgresStore` with distributed locking when several RoomKit
workers must share conversations.

## Quick Start

```python
from roomkit import RoomKit, SQLiteStore

store = SQLiteStore("roomkit.db")
kit = RoomKit(store=store)

room = await kit.create_room(room_id="support-1")

# RoomKit closes the store it owns during framework shutdown.
await kit.close()
```

The default path is `roomkit.db`. Parent directories are created when the
store first opens:

```python
store = SQLiteStore("var/data/roomkit.db")
```

## What Is Stored

Rooms, events, bindings, participants, identities, tasks, observations, read
markers, and idempotency keys persist across restarts. Pydantic models are
stored as JSON for exact round-tripping, with queryable fields mirrored into
indexed columns.

Event indexes use a durable per-room high-water mark. Deleting history never
reuses an old index, and database constraints reject duplicate room indexes or
idempotency keys. These invariants are enforced by SQLite transactions, not by
application timing.

## Full-Text Search

Message text is mirrored into an FTS5 index. `search_events()` is an
SQLite-specific extension to the common store contract:

```python
results = await store.search_events("invoice overdue", room_id="support-1")

for event in results:
    print(event.content)
```

Search terms are escaped and combined as an AND query. Empty or punctuation-
only input returns an empty list.

## Lifecycle and Schema Safety

`await kit.close()` closes the store. If you use a store without `RoomKit`,
close it directly; `close()` is idempotent:

```python
store = SQLiteStore("roomkit.db")
try:
    room = await store.get_room("support-1")
finally:
    await store.close()
```

RoomKit records a schema version in the database. Known older files are
migrated transactionally. A file created by a newer RoomKit version raises
`SQLiteSchemaError` instead of being silently relabelled or partially opened:

```python
from roomkit import SQLiteSchemaError
```

Back up the database before upgrading a production deployment, as with any
persistent store.

## Single-Process Boundary

SQLite can serialize writes from multiple connections, but a RoomKit inbound
turn includes decisions and side effects outside one database transaction.
`InMemoryLockManager` coordinates those steps only inside the current process.
RoomKit therefore warns when `SQLiteStore` is paired with the default lock
manager: the combination is supported for one process, not for a load-balanced
worker fleet.

For horizontal scaling, use the [PostgreSQL store](postgres-store.md) together
with [distributed locking](scaling.md).

## Data Security

SQLite does not encrypt the file. Treat it as conversation data:

- restrict filesystem permissions to the application account;
- keep the database and its `-wal`/`-shm` companions out of source control;
- encrypt the volume or host when content-at-rest encryption is required;
- protect and test backups;
- never place credentials or provider secrets in room metadata or message
  content.
