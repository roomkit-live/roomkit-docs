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
from roomkit.store import PostgresStore

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


def build_agent(channel_id: str) -> Agent | None:
    if channel_id == "meteo":
        return Agent("meteo", provider=my_provider, tools=[forecast], tool_handler=handle)
    return None


backend = RedisTurnBackend("redis://localhost:6379")
executor = TurnExecutor(build_agent, hosts={"meteo"})
await backend.serve(executor)
```

The executor needs only the Redis server: the store stays with the room, which
sends what a turn reads, and a hook on the executor asks the room for its
context. Run as many executor processes as you need: a room's turns stay
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

An entry ending in `*` declares a prefix: `hosts={"acme-*"}` takes every
agent whose id starts with `acme-`, and `hosts={"*"}` every agent. That is
how a pool of generic executors serves agents created after it started: each
executor builds an agent from its configuration, and is never told of each
one. An executor builds an agent at its first turn, channel call or committed
event there, never for a room's attach or detach, and keeps it until it
closes or the kit releases the channel (`kit.release_channel`).

`max_channels` bounds what one executor keeps built:

```python
executor = TurnExecutor(build_agent, hosts={"acme-*"}, max_channels=200)
```

Past the bound, the executor lets go of the agent it used least recently,
never one something runs on (a turn, a channel call, a committed event, a
room's attach or detach), as a release lets it go: the state it kept for its
rooms goes (an ACP session reopens at its next turn, a speak policy listens
again), and its next turn builds it anew, the room sending what that turn
reads. An agent built only to hand it
a committed event is the first let go. No bound by default: size it above
the agents one executor serves at once.

The executor closes each agent it lets go of (its bound, a release, its own
close), and an agent closes its provider with it. Give each build a provider
of its own: a provider shared by the agents a factory builds is closed with
the first one let go, and every other agent's next call fails.

```python
def build_agent(channel_id: str) -> Agent | None:
    config = agent_configs.get(channel_id)  # your service's configuration
    if config is None:
        return None
    return Agent(channel_id, provider=config.provider, system_prompt=config.prompt)


executor = TurnExecutor(build_agent, hosts={"acme-*"})
await backend.serve(executor)
```

A turn or channel call of an agent the factory builds nothing for (it returns
`None`) fails with `no_executor`, as when no executor declares the agent, and
is not handed to another executor. The room's next turns go to that executor
too, while it lives: every executor declaring a prefix must build every agent
under it, from the same configuration. `examples/turns_executor_pool.py` runs
two pool processes, agents created on the fly, and one pool killed mid-way.

Upgrade every room owner process before you start executors that declare
prefixes: an older owner reads `acme-*` as an agent's id, and fails the turns
of `acme-sales` with `no_executor`.

The prefix is the only scope an executor declares: `hosts={"*"}` takes the
turns of every declared agent, whatever its room or organization, from every
process sharing the backend's `key_prefix`. Assign agent ids yourself, a
prefix per pool you want kept apart, and never let a tenant choose one.

Room hooks are part of the declaration: an executor that does not define a
hook a room enabled (`executor.room_hook("audit", ...)`,
`await kit.enable_room_hook(room_id, "audit")`) is never given that room's turns.
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

## Which agents can answer

The kit reads the executors from the same beats placement reads:

```python
for info in await kit.executors():
    print(info.executor_id, info.node, sorted(info.hosts or []), info.last_beat, info.online)

for presence in await kit.agent_presence(room_id):
    print(presence.channel_id, presence.online, presence.executors)
```

An executor is online while it beats within `heartbeat_timeout`: a process
killed stays online until that bound passes, never offline on one late beat.
An agent is online in a room while an executor online covers it there: it
declares the agent and defines the room's enabled hooks.

The kit also publishes each agent's presence in its rooms, as the presence
ephemerals people's presence uses (`presence_online` / `presence_offline`,
`user_id` the agent's channel id): as the agent is attached, and when the
executors covering it change, in its active and paused rooms (the 10,000
oldest at most). Every room owner process sees the change; one publishes it,
the presence last published being kept in Redis
(`TurnBackend.claim_presence`), so a restart publishes nothing that did not
change. A publication, or a claim, that fails is tried again at a later look
of the owner that made it (an owner closing lets go of the claims it could not
publish), and a presence failure never fails the attach or the detach that
caused it. `DiscussionConsole` shows an agent no executor serves as `offline`.
`examples/agent_presence.py` runs it in one process.

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
executor = TurnExecutor(build_agent, hosts={"meteo"}, error_classes=ERRORS)
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
- **Several owner processes** need a store they share (`PostgresStore`) and the
  distributed lock manager (see [Horizontal Scaling](scaling.md)). Executors
  need neither: they hold no store and take no room lock.

## Runnable example

`examples/turns_redis.py` runs the owner in one process and starts the
executor in a second one; it kills the executor mid-answer, shows the turn
failing with `executor_lost`, and starts a new executor that takes the room's
next turn. It uses Claude Haiku when `ANTHROPIC_API_KEY` is set, a scripted
model otherwise.

```bash
REDIS_URL=redis://localhost:6379 uv run python examples/turns_redis.py
```
