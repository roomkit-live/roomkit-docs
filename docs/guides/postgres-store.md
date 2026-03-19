# PostgreSQL Storage

`PostgresStore` is the production-ready `ConversationStore` backend for RoomKit. It uses asyncpg for connection pooling and JSONB for flexible model storage with full transaction support.

## Installation

```bash
pip install roomkit[postgres]
```

This installs `asyncpg>=0.29` as an optional dependency.

## Quick Start

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.store.postgres import PostgresStore

store = PostgresStore("postgresql://user:pass@localhost/roomkit")
await store.init(min_size=2, max_size=10)

kit = RoomKit(store=store)
```

Or with the async context manager (handles init and close automatically):

```python
async with PostgresStore("postgresql://user:pass@localhost/roomkit") as store:
    kit = RoomKit(store=store)
    # ... use kit ...
# store.close() called automatically
```

### Environment Variable DSN

```python
import os
store = PostgresStore(os.environ["DATABASE_URL"])
```

## Connection Pooling

```python
await store.init(
    min_size=2,    # Minimum warm connections (default: 2)
    max_size=10,   # Maximum concurrent connections (default: 10)
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `min_size` | `2` | Minimum connections kept warm |
| `max_size` | `10` | Maximum concurrent connections |

Connection acquisition timeout is 5 seconds. If all connections are in use and the pool is at max_size, requests wait up to 5s before raising an error.

### Pre-Built Pool

For advanced use (sharing a pool across services):

```python
import asyncpg

pool = await asyncpg.create_pool("postgresql://...", min_size=5, max_size=20)
store = PostgresStore(pool=pool)
# Store won't close the pool on exit (caller is responsible)
```

## Schema

PostgresStore creates 10 tables on first `init()`:

| Table | Purpose | Key Indexes |
|-------|---------|-------------|
| `rooms` | Room state and metadata | status, org_id, metadata (GIN) |
| `events` | Conversation events (JSONB) | room_id+created_at, idempotency_key |
| `bindings` | Channel attachments | channel_id |
| `participants` | Room participants | (room_id, id) PK |
| `identities` | User identities | id PK |
| `identity_addresses` | Multi-channel address lookup | (channel_type, address) unique |
| `tasks` | Background tasks | room_id |
| `observations` | AI/ML observations | room_id |
| `read_markers` | Read state tracking | (room_id, channel_id) PK |
| `schema_version` | Migration tracking | version |

All Pydantic models are stored as JSONB in a `data` column — flexible and queryable. Foreign keys use `ON DELETE CASCADE` for automatic cleanup when rooms are deleted.

### Schema Initialization

The schema is idempotent — `CREATE TABLE IF NOT EXISTS` and `INSERT ... WHERE NOT EXISTS` patterns ensure safe re-runs. Version tracking via `schema_version` table enables future migrations.

## Room Operations

```python
from __future__ import annotations

from roomkit.models.room import Room

# Create
room = await store.create_room(Room(id="room-1", organization_id="org-1"))

# Read
room = await store.get_room("room-1")

# Update
room.status = "paused"
await store.update_room(room)

# Delete (cascades to events, bindings, participants, tasks, observations)
await store.delete_room("room-1")

# List with pagination
rooms = await store.list_rooms(offset=0, limit=50)

# Find by criteria (supports JSONB metadata filtering)
rooms = await store.find_rooms(
    organization_id="org-1",
    status="active",
    metadata_filter={"tier": "premium"},
    limit=100,
)

# Find latest room for a participant
room = await store.find_latest_room("user-123", channel_type="sms", status="active")

# Reverse lookup: find room by channel
room_id = await store.find_room_id_by_channel("sms-main", status="active")
```

## Event Indexing

Events are indexed with monotonically increasing per-room indices. The `add_event_auto_index()` method assigns indices atomically:

```python
# Atomic: lock room → get next index → insert event (single transaction)
event = await store.add_event_auto_index("room-1", event)
print(event.index)  # 0, 1, 2, ...
```

This uses `SELECT ... FOR UPDATE` on the rooms table to serialize concurrent index assignments, preventing collisions even under high concurrency.

### Event Queries

```python
# Paginated timeline
events = await store.list_events("room-1", offset=0, limit=50)

# With visibility filter
events = await store.list_events("room-1", visibility_filter="all")

# Idempotency check (prevent duplicate processing)
exists = await store.check_idempotency("room-1", "msg-unique-key")

# Event count
count = await store.get_event_count("room-1")
```

## Identity Storage

PostgresStore supports multi-channel identity resolution:

```python
from __future__ import annotations

from roomkit.identity.base import Identity

# Create identity with addresses (single transaction)
await store.create_identity(Identity(
    id="user-1",
    display_name="Alice",
    channel_addresses={
        "sms": ["+1234567890"],
        "email": ["alice@example.com"],
    },
))

# Resolve by channel address
identity = await store.resolve_identity("sms", "+1234567890")

# Link additional address (transactional)
await store.link_address("user-1", "whatsapp", "+1234567890")
```

Address uniqueness is enforced by a `(channel_type, address)` unique constraint — one address can only belong to one identity.

## Read Tracking

Efficient unread counting using event indices:

```python
# Mark specific event as read
await store.mark_read("room-1", "ws-alice", "event-42")

# Mark all events as read (transactional: fetch latest + update marker)
await store.mark_all_read("room-1", "ws-alice")

# Count unread events (uses index comparison — O(1) lookup)
count = await store.get_unread_count("room-1", "ws-alice")
```

## Tasks and Observations

```python
from roomkit.models.events import Task, Observation

# Background tasks
task = await store.add_task(Task(id="task-1", room_id="room-1", status="pending"))
tasks = await store.list_tasks("room-1", status="pending")
task.status = "completed"
await store.update_task(task)

# AI/ML observations
obs = await store.add_observation(Observation(id="obs-1", room_id="room-1", data={"sentiment": 0.8}))
observations = await store.list_observations("room-1")
```

## Telemetry

All operations are instrumented with `SpanKind.STORE_QUERY` telemetry spans:

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.store.postgres import PostgresStore
from roomkit.telemetry import OpenTelemetryProvider

store = PostgresStore("postgresql://...")
kit = RoomKit(
    store=store,
    telemetry=OpenTelemetryProvider(service_name="roomkit"),
)
# All store operations now emit spans with:
#   Attr.STORE_OPERATION (e.g., "create_room", "add_event")
#   Attr.STORE_TABLE (e.g., "rooms", "events")
```

## Production Tips

### Connection Pool Sizing

- **min_size**: Set to your baseline concurrency (e.g., 2-5)
- **max_size**: Set to your peak concurrency (e.g., 10-50)
- Monitor pool utilization — if requests frequently wait for connections, increase max_size

### Monitoring

```python
# Check pool stats (asyncpg)
pool = store._pool
print(f"Pool size: {pool.get_size()}")
print(f"Free connections: {pool.get_idle_size()}")
```

### Cascading Deletes

Deleting a room automatically cascades to all child tables (events, bindings, participants, tasks, observations, read markers). This is safe and atomic.

### Backup Strategy

Since all models are stored as JSONB, standard PostgreSQL backup tools (`pg_dump`, continuous archiving) work out of the box. The JSONB format also supports partial indexing and GIN indexes for metadata queries.

## Migration from InMemoryStore

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.store.postgres import PostgresStore

# Before (development):
kit = RoomKit()  # Uses InMemoryStore by default

# After (production):
store = PostgresStore("postgresql://user:pass@db/roomkit")
await store.init(min_size=2, max_size=20)
kit = RoomKit(store=store)
```

The `ConversationStore` ABC ensures both implementations expose the same API. No other code changes needed.
