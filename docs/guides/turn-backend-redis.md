# Agent Turns in Another Process (Redis)

Every turn of an intelligence channel runs through a turn backend (RFC
§10.6). With the default in-memory backend the turn runs in the kit's
process. With `RedisTurnBackend` it runs on an executor in another process,
or on another machine: the CPU an agent spends (its memory, building its
context, its tool loop) leaves the process that serves your HTTP and your
WebSockets, while the kit keeps the room. Ordering, the delivery cursor, the
commits and the broadcasts stay where they were; for the room, a turn
elsewhere is an `on_event` that returns a stream.

```bash
pip install roomkit[redis]
docker run --rm -d -p 6379:6379 redis:7
```

## Two processes

The room owner's process holds the agent by its declaration alone: it knows
what the agent is, not its provider, its MCP clients or its memory.

```python
from roomkit import AgentIdentity, ChannelDeclaration, RedisTurnBackend, RoomKit
from roomkit.store.postgres import PostgresStore

kit = RoomKit(
    store=PostgresStore(dsn=DSN),
    turn_backend=RedisTurnBackend("redis://localhost:6379"),
)
kit.register_channel(
    ChannelDeclaration(channel_id="meteo", identity=AgentIdentity(name="Meteo"))
)
```

The executor's process builds the agent and runs its turns:

```python
from roomkit import Agent, RedisTurnBackend, TurnExecutor
from roomkit.store.postgres import PostgresStore


def build_agent(channel_id: str) -> Agent | None:
    if channel_id == "meteo":
        return Agent("meteo", provider=my_provider, tools=[forecast], tool_handler=handle)
    return None


backend = RedisTurnBackend("redis://localhost:6379")
executor = TurnExecutor(build_agent, hosts={"meteo"}, store=PostgresStore(dsn=DSN))
await backend.serve(executor)
```

Both processes share the store (the executor reads the rooms there) and the
Redis server. Run as many executor processes as you need: a room's turns stay
on one of them, new rooms spread over them. With an `OpenTelemetryProvider`
in each process (`TurnExecutor(telemetry=...)` on the executor's side), a
turn shows as one trace across both ([Telemetry](telemetry.md#one-trace-across-processes)).

## What an executor declares

`hosts={"meteo"}` is the set of channels the executor runs. Only an executor
declared with a set takes turns from other processes; one given a predicate
(`hosts=lambda channel_id: ...`) takes the turns of its own process only, as
the kit's own executor does for the agents you register live. An executor
with no `hosts` would take the turns of every channel the kit does not run
itself: always pass the set.

Room hooks are part of the declaration: an executor that does not define a
hook a room enabled (`executor.room_hook("audit", ...)`,
`kit.enable_room_hook(room_id, "audit")`) is never given that room's turns.
On an executor a name carries one hook per trigger, a redefinition replacing
it, and the name a room enables joins the room's hooks: a room hook of the
room under that name refuses it (`ValueError`).

## Where a turn goes

An agent you registered live on the kit runs on the kit's own executor alone,
whatever else declares it; a turn that executor does not take (a room hook it
does not define) fails with `no_executor`. For the other channels the room
owner chooses, from the executors alive and what they declare:

1. the executor that took the room's last turn of the channel, while it
   lives and takes it: its caches and its session state stay (an ACP
   session, a speak policy's listening state);
2. else an executor of the owner's own process, carried in memory;
3. else one alive elsewhere, rooms no executor holds yet spread over them.

A turn no executor declares fails at once with `no_executor`. Which executor
holds a room is kept in Redis, so every owner process places the room's turns
the same way.

## Liveness and failures

An executor's process beats from a thread of its own, on a Redis connection
of its own: a turn that keeps its event loop busy (the case the backend
exists for) is slow, never lost. The room owner counts those beats with the
turn's messages.

That is also what frees the owner's process. Measured with Claude Haiku and a
tool that burns one second of CPU: with the agent in the kit's process the
kit's event loop stalls for the whole second (every HTTP request and
WebSocket waits); with the agent on an executor over Redis, the kit's loop
never stalls more than a few milliseconds.

| What happens | What the room sees |
|---|---|
| The executor's process is killed mid-answer | The turn fails with `executor_lost` within `heartbeat_timeout`; the text already shown is stored, as a response that failed (RFC §12.2 step 13s) |
| No executor takes the turn within `accept_timeout` | The turn fails (`executor_lost`) |
| The executor closed but its process still runs | Its process refuses the turn at once, and the turn goes to another executor (or fails with `no_executor` when none takes it) |
| The executor is restarted under the same id | It is another executor: the turns of the dead process are lost, never kept alive by the new one |
| A process serves a new executor under the id of one it closed (a reload) | The new one takes the turns from then on |
| The executor's process closes right after a turn ended | The turn's end, still queued, is written before the process goes (within `cancel_timeout`) |
| The executor's process closes mid-answer | The process cuts the turn and writes its end: the turn fails with `executor_lost` within `cancel_timeout`, not `heartbeat_timeout` |
| The executor's loop stalls right after a turn ended, before its end is written | The process still names the turn running until its end is written, so the owner waits for it and the turn completes |
| A turn reaches an executor whose process is closing | That process refuses it, and the turn goes to another executor |
| Redis refuses a write during a turn | That turn fails within the heartbeat window, never waits for good |
| Redis goes silent without closing a connection (a NAT or a load balancer forgot it) | The process stays among the living and its turns go on while `heartbeat_interval` is under half of `heartbeat_timeout`: the beat and the inbox's read give up within that half and try again at once on a new connection; a turn's message whose write was cut fails that turn |
| The room owner goes away | The executor cuts the turn as `owner_lost` |
| The kit's own executor is closed while the kit runs (`kit.turn_executor.close()`) | A live agent's turn fails with `no_executor`: it never goes to a host executor's copy, which would run without the live agent's state |
| The room owner closes mid-answer (the kit's `close()`) | The turn fails on the owner with `backend_closed`: its backend closed, its executor is not lost (never `executor_lost`); the executor cuts the turn as `owner_lost` |

In every case the room goes on: the next message is delivered, and its turn
goes to an executor alive.

```python
from roomkit import TurnTimeouts

timeouts = TurnTimeouts(heartbeat_interval=2.0, heartbeat_timeout=10.0, accept=10.0)
backend = RedisTurnBackend(url, timeouts=timeouts)   # the same in every process
```

The backend's timeouts are those of the kit it is given to: give every
process, and every executor's `TurnExecutor(timeouts=...)`, the same (the
backend warns when an executor it serves keeps others).

A request reaches one executor at most: an executor's process acknowledges
it, a refusal places it again, and a process that does not answer within
`accept_timeout` loses the turn, which is never run twice. A client that
retries a pipeline whose connection dropped after Redis executed it writes
its messages twice: each is still handled once.

## Errors across processes

An error a turn raised on the executor reaches the room owner rebuilt from a
message (RFC §10.6.8). RoomKit's classes and Python's built-in ones come back
as themselves; your application's come back as themselves only in a process
that declares them, and each process declares its own: the owner reads what
the executor raised, and the executor reads what an owner-served tool or a
host verb raised.

```python
ERRORS = (QuotaExceededError, httpx.HTTPError)  # a class covers its subclasses

# The room owner
kit = RoomKit(store=store, turn_backend=backend, error_classes=ERRORS)
# Each executor's process
executor = TurnExecutor(build_agent, hosts={"meteo"}, store=store, error_classes=ERRORS)
```

A class a process does not declare comes back as a `RemoteTurnError`: its
`kind` names the class, its attributes JSON carries are kept, and it is also
an instance of the nearest ancestor the process rebuilds, so a subclass of
`ProviderError` still reads as a `ProviderError` (its log line, `ON_ERROR`).
A class named in a message is looked up among the modules the process has
already loaded: a message never makes a process import anything.

## Work after a turn

A thought posted after the turn ended, a background task's status: what an
executor sends through `current_turn_host()` once its turn returned reaches
the room owner process that submitted that turn, even with several owner
processes on the same Redis. That work starts fresh once its turn is over: an
event it sends is at the chain depth it names, not the turn's.

## Security

Redis is the trust boundary. The executor ids and owner addresses its
messages carry are not authenticated beyond it: whoever can write to the
Redis server can act as an executor or an owner, and whoever can read it
reads the rooms' turns. Restrict it as the rooms it carries: Redis ACLs, TLS
(`rediss://`), a network only your processes reach. Use a `key_prefix` per
deployment.

## Connecting

`RedisTurnBackend(url)` connects with a pool its callers wait on when every
connection is in use. To use a client of your own (a sentinel, custom TLS),
pass both: `RedisTurnBackend(url, client=client)`; the process's beat thread
connects with `url`, so it must name the same server.

Every connection the backend opens gives up a call after half of
`heartbeat_timeout` (5 s by default); the inbox's blocking read waits the time
Redis holds it more, a second at most and never over `heartbeat_interval`.
The beat or the read it cut is tried again at once on a new connection, so
while `heartbeat_interval` stays under half of `heartbeat_timeout` (the
defaults: 2 s and 10 s), a Redis gone silent without closing the connection
costs one call, never the process's place among the living. A client you pass
keeps its own settings, and the backend builds no pool beside it: give it a
`socket_timeout` of the same order (redis-py before 8.0 sets none). Options in
the URL's query (`?socket_timeout=...`) take precedence over the backend's,
the inbox's connection included: one shorter than the inbox's blocking read
fails each of its idle reads.

## Prerequisites

- **Redis 6.2 or later.**
- **redis-py 8.0.1 or later on a unix socket.** redis-py 8.0.0's sync
  client, which the process's beat thread uses, fails the first call of each
  new `unix://` connection (`ValueError: Cannot enable maintenance
  notifications...`) and keeps that connection without selecting the URL's
  database: one beat is lost per connection, the immediate retry after a cut
  among them, and with a database other than 0 every beat lands in database
  0, so the other processes take the process for lost. 5.0.2 to 7.4 work too;
  over TCP every version from 5.0.2 does.
- **A shared store**: `PostgresStore` (or `SQLiteStore` on one machine). An
  executor that takes turns from other processes without one logs a warning.
- **Several owner processes** need the distributed lock manager as well (see
  [Horizontal Scaling](scaling.md)); executors take no room lock.

## Runnable example

`examples/turns_redis.py` runs the owner in one process and starts the
executor in a second one; it kills the executor mid-answer, shows the turn
failing with `executor_lost`, and starts a new executor that takes the room's
next turn. It uses Claude Haiku when `ANTHROPIC_API_KEY` is set, a scripted
model otherwise.

```bash
REDIS_URL=redis://localhost:6379 uv run python examples/turns_redis.py
```
