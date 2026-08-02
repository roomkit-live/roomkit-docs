# Status Bus

Share real-time status updates between agents in a multi-agent setup. When an execution agent completes a task, the voice agent is notified immediately — no polling needed.

## Quick start

```python
from roomkit import RoomKit
from roomkit.orchestration.status_bus import StatusBus

# Create a bus (in-memory with optional JSONL persistence)
bus = StatusBus(persist_path="/tmp/session.jsonl")

# Any agent can post status updates
bus.post("exec", "search_google", "ok", detail="Found 7 results")
bus.post("exec", "click_result", "ok", detail="Clicked RoomKit link")
bus.post("system", "task_completed", "completed", detail="Done in 48s")

# Any agent can read recent activity
entries = await bus.recent(5)
text = await bus.recent_text(5)
# → [19:05:15] exec: search_google → ok | Found 7 results
# → [19:05:20] exec: click_result → ok | Clicked RoomKit link
# → [19:05:22] system: task_completed → completed | Done in 48s

# Subscribe for real-time notifications
async def on_status(entry):
    if entry.status == "completed":
        print(f"Task done: {entry.detail}")

await bus.subscribe(on_status)
```

## Framework integration

The `StatusBus` is wired into `RoomKit` automatically — no manual setup needed:

```python
from roomkit import RoomKit
from roomkit.orchestration.status_bus import StatusBus

# Default in-memory bus (always available)
kit = RoomKit()
kit.status_bus.post("exec", "search_google", "ok", detail="Found 7 results")

# Custom bus with persistence
kit = RoomKit(status_bus=StatusBus(persist_path="/tmp/session.jsonl"))
```

### Framework events

Every status post emits a `status_posted` framework event:

```python
@kit.on("status_posted")
async def on_status(event):
    entry = event.data  # dict with ts, agent_id, action, status, detail, metadata
    if entry["status"] == "completed":
        print(f"Task done: {entry['detail']}")
```

The bus is closed automatically when `kit.close()` is called.

## How it works

The `StatusBus` is a shared event log with pub/sub. All agents in a room write to it, and any agent can subscribe to be notified when new entries arrive.

```
Voice Agent ──post──→ StatusBus ──notify──→ Exec Agent
Exec Agent  ──post──→ StatusBus ──notify──→ Voice Agent
System      ──post──→ StatusBus ──notify──→ All subscribers
```

### Status entry

Each entry has:

| Field | Type | Description |
|-------|------|-------------|
| `ts` | str | ISO timestamp |
| `agent_id` | str | Who posted (e.g. "exec", "voice") |
| `action` | str | What happened (e.g. "search_google", "click_result") |
| `status` | str | Outcome: `ok`, `failed`, `pending`, `info`, `completed` |
| `detail` | str | Human-readable description |
| `metadata` | dict | Optional structured data |

### Sync and async posting

```python
# Sync (fire-and-forget, safe from any context)
bus.post("exec", "search_google", "ok", detail="Found results")

# Async (awaits subscriber notification)
await bus.post_async("exec", "search_google", "ok", detail="Found results")
```

## Pluggable backends

The default `InMemoryStatusBackend` works for single-process setups. For
distributed deployments, use `RedisStatusBackend` — entries live in a capped
Redis list (shared history) and fan out to every process via pub/sub:

```python
from roomkit.orchestration import RedisStatusBackend
from roomkit.orchestration.status_bus import StatusBus

bus = StatusBus(backend=RedisStatusBackend("redis://localhost:6379"))
```

Requires `pip install roomkit[redis]`. Options:

```python
RedisStatusBackend(
    "redis://localhost:6379",
    key_prefix="roomkit:status",  # list key + pub/sub channel namespace
    max_entries=500,              # shared history cap (as InMemoryStatusBackend)
)
```

Semantics that differ from the in-memory backend:

- `publish()` returning does **not** mean subscribers ran — local callbacks
  are notified through the Redis round-trip, asynchronously.
- Every process's subscribers observe every entry from every process. With
  RoomKit that means each worker re-emits all entries as `status_posted`
  framework events — distributed observability by default.
- Pub/sub notifications published while a process is disconnected are lost,
  but the shared history stays consistent — consumers can resync with
  `recent()`.

For another transport (NATS, ...), implement the `StatusBackend` ABC:
`publish()`, `recent()`, `subscribe()`, `unsubscribe()`, and optionally
`close()`. See `examples/realtime_redis.py` for a cross-process demo.

## Integration with voice agents

Auto-inject status updates into a `RealtimeVoiceChannel` so the voice agent knows what the execution agent is doing:

```python
async def on_exec_status(entry):
    if entry.agent_id == "exec" and entry.status in ("info", "completed"):
        for session in voice_channel.get_room_sessions("my-room"):
            await voice_channel.inject_text(
                session,
                f"[Agent update] {entry.action}: {entry.detail}",
                role="user",
                silent=True,
            )

await bus.subscribe(on_exec_status)
```

## Delegation guard

Prevent re-delegation when a task was just completed:

```python
completed = await bus.recent(1, status="completed")
if completed:
    age = (now - completed[0].ts).total_seconds()
    if age < 5:
        return "Task already completed. Ask the user if they need something different."
```

## JSONL persistence

Pass `persist_path` to persist entries to a JSONL file:

```python
bus = StatusBus(persist_path="/tmp/screen_ai/session.jsonl")
```

Each entry is appended as one JSON line. Useful for debugging and audit trails.

## Print summary

```python
await bus.print_summary()
```

```
Status Log
============================================================
   1. [+] exec   search_google("roomkit conversation AI")
      Found 15 text elements on results page
   2. [+] exec   click_result(118)
      Clicked "roomkit.live" at (274,313)
   3. [*] system task_completed
      Duration: 48303ms

  Total: 3 entries (2 ok, 0 failed, 1 completed)
```
