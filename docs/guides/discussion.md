# Discussion: Agents in a Group Chat

Several agents and one or more people hold one conversation in a room. Any
agent may address any other with `@channel_id`, a person may address any
agent, and nobody fixes the order in advance: who speaks next follows from who
was addressed. One agent speaks at a time (RFC §19.7.5); long work runs as
tasks, apart from the conversation. An agent is an AI channel or an external
agent over ACP, such as Claude Code.

```python
from roomkit import Agent, Discussion, RoomKit

investigator = Agent("investigator", provider=..., role="Investigator",
                     description="Reads logs and metrics first")
dev = Agent("dev", provider=..., role="Developer",
            description="Knows the code and the releases")
sre = Agent("sre", provider=..., role="Site reliability engineer",
            description="Runs production changes")

kit = RoomKit()
room = await kit.create_room(orchestration=Discussion(
    agents=[investigator, dev, sre],
    everyone=["investigator", "dev", "sre"],
    people=["oncall"],
))
```

A runnable version, with scripted models and no keys, is
`examples/discussion_group_chat.py`.

## Why a strategy

A room of several agents left to the default policy dogpiles: every agent
answers the same message at once, each reading the room as it was before the
others answered, and every answer solicits the others again.
`AgentResponsePolicy.ADDRESSED_ONLY` stops the dogpile by stopping agents from
answering each other at all. A discussion keeps them talking and takes the
turns itself.

[Swarm](orchestration.md#swarm) passes the conversation from one agent to the
next: the active agent talks with the user and the others wait for a handoff.
In a discussion every agent stays in the conversation and speaks when it is
addressed, the way people in a group chat do.

## Who speaks next

The strategy keeps a **speak queue** per room. A committed message queues the
agents it asks for; the strategy gives the next turn once the current one has
ended.

| Message | Asks |
|---------|------|
| A person's message naming agents (`@dev @sre ...`) | Those agents, at the front of the queue, in the order named |
| A person's message naming nobody | The agents that asked that person, else `everyone`, in its order; with a [dispatch policy](#who-takes-a-message-that-names-nobody), the agents it picks |
| A person's message naming only people | Nobody |
| An agent's answer naming agents | Those agents, at the back, once the turn has ended |
| Any other sender's message (a bot, a webhook, `kit.deliver()`) | The agents its address names, at the back |

- `@all` names every agent. A name holds letters of any script, digits,
  `_ . -`. Names are read in what the speaker wrote once every host hook has
  run, never in a fenced code block or a quoted (`>`) line, and they set the
  event's `addressed_to`. A name never gets around visibility: an agent that
  cannot read the message is not queued by it.
- An agent named several times while queued is owed **one** turn; it answers
  the latest person's message among the events that asked for it.
- The agent that just spoke does not take the next turn while another agent of
  the queue can.
- `everyone`'s order is yours: put first the agent that should look at the
  facts before the others act. The discussion does not rotate it.

### Rooms where people talk among themselves

With `addressed_only=True`, a person's message asks only the agents it names:
people who talk together in the room are never cut into by an agent. To let
agents judge for themselves whether an unaddressed message is for them, leave
`addressed_only` off and give them a [speak policy](speaking-turns.md), or give
the discussion a dispatch policy, which judges once for the room (below).

## Who takes a message that names nobody

A person's message that names no agent goes to the agents that asked that
person a question, else to `everyone`, one turn each. In a room of
specialists most of those turns end in `(silent)`, the agent that should
answer may come third, a "thanks" wakes the whole team, and an agent that
just asked "@ops did anyone change a flag?" takes your next message even when
it is "Customers are writing to support, what are they saying?". A
**dispatch policy** decides instead which agents take it, in which order, or
that none does (RFC §19.7.5 rule 18):

```python
from roomkit import Agent, ClassifierDispatchPolicy, Discussion, JevClassifier

team = [
    Agent("investigator", provider=..., name="Investigator",
          description="reads checkout-api's logs and request traces"),
    Agent("sre", provider=..., name="SRE",
          description="reads production metrics and the deploy history, can roll back"),
    Agent("dev", provider=..., name="Developer",
          description="reads the feature-flag history and the source code"),
    Agent("comms", provider=..., name="Comms",
          description="reads the support tickets, owns the public status page"),
]
await kit.create_room(orchestration=Discussion(
    team,
    dispatch=ClassifierDispatchPolicy(JevClassifier(), threshold=0.5, max_agents=2),
))
```

`ClassifierDispatchPolicy` asks a [classifier](classifiers.md), in one call,
whether each agent should take the message, from each agent's name and
description, the recent conversation and the message. The agents whose
probability reaches `threshold` take it, likeliest first, at most
`max_agents`; none reaching it, no agent does.

On eight unaddressed messages to these four agents
(`examples/discussion_dispatch.py`), the same six answers came from:

| | Turns | Silent turns | Time per decision |
|---|---|---|---|
| No policy (`everyone`) | 32 | 26 | — |
| Jev (TypeSafe) | 8 | 2 | 107–280 ms |
| Claude Haiku (`LLMClassifier`) | 7 | 1 | 1.2–2.9 s |

What a policy decides, and what it cannot:

- **Only a message that names nobody.** A name is the person's word and is
  never decided. `addressed_only` and a dispatch policy are exclusive (the
  constructor refuses both).
- **Knowing who waits for an answer.** The agents that asked the person come
  first among the candidates and are listed in `DispatchTurn.asked`;
  `ClassifierDispatchPolicy` marks them, so "eu-west" goes to the agent that
  asked which region, and a new request goes to whoever it is for (10 right
  out of 10 on Jev with the mark, 9 without).
- **Among the candidates.** They are the agents that asked the person, then
  `everyone`, in its order, less the agents the message does not reach
  (visibility, access) and those that only listen. A decision naming anyone
  else is cut down to them.
- **Once, at the message's place.** The process holding the lease decides off
  the room lock, before it gives another turn. The agents picked go to the
  front in the order decided, still before the agents a later message asks
  for. Each still answers through its own speak policy, if it has one.
- **Never silencing the room.** A policy that raises, takes longer than
  `dispatch_timeout` (5 s) or returns no readable decision leaves the
  message to the rule without a policy (the agents that asked the person,
  else every candidate), reported with the reason `fallback`. At most 16
  messages wait for a decision (none is decided while no process that
  installed the discussion is alive); past them, the oldest goes the same
  way.
- **Not stored with the message.** Its `addressed_to` stays null, and a
  regenerated answer with none left asks `everyone`.

Every decision applied fires `ON_DISPATCH_DECISION`, once per message, with a
`DispatchDecisionEvent`:
the message, the candidates, the `DispatchDecision` (`agents`, `reason`,
`judgments`) and `duration_ms`.

```python
@kit.hook(HookTrigger.ON_DISPATCH_DECISION, execution=HookExecution.ASYNC)
async def log_dispatch(event: DispatchDecisionEvent, ctx) -> None:
    logger.info("%s -> %s (%s, %d ms)", event.event.id, event.decision.agents,
                event.decision.reason, event.duration_ms)
```

A policy of your own implements `DispatchPolicy.decide(turn)`: a
`DispatchTurn` carries the message, the conversation before it that a
candidate may read, who said what (`speakers`) and the candidates with their
identity (`DispatchCandidate`), and which of them wait for the person's
answer (`asked`). `MockDispatchPolicy` scripts decisions for tests.

## What a turn reads

Each turn is a rerun of the event it answers, planned for one agent and run in
the room's delivery lane. Its context is built when the turn starts, so what
was said while the agent waited is in it, and the event it answers keeps its
place in the history instead of being read as the last message. The turn's
notes tell the agent who else is in the room (their channel ids, with the
`name`, `role` and `description` of Agents and of external agents given one),
whom it may address, who asked for this turn, and how the room works.

A speak policy is consulted on a turn as on any event the agent would answer.
Without one, the model may stay silent itself by answering the silence token
(`silent_token`, default `"(silent)"`, read ignoring case, a final period and
its brackets). That row is stored `BLOCKED` with
`blocked_by="discussion_silent"`, delivered nowhere and read for no name. A
streamed answer is held back while its start could still be the token, so a
live transport never sees it.

`process_inbound()` returns once the message is committed and broadcast, with
no agent's answer in its result: the answers come in the turns that follow.

Each agent's memory provider is still handed every message the agent may see
(not its own), as it commits, as in a room with no discussion: a memory that
learns as messages arrive (a retrieval index, a summary) learns the whole
conversation, not only the messages that asked for the agent.

## Waiting for a person

When an agent names a person (`@oncall may I roll back?`), the discussion
records that it asked them. When no agent of the queue can take a turn, the
discussion **waits** for a person if it owes one an answer, if an agent of the
queue only listens, or if the depth limit stopped the next turn. A person's
next message ends the wait; naming nobody, it answers the agents that asked
that person.

Agents address people by the names the transcript gives them: a participant's
name, the name a transport stamped (`sender_name`), else the channel of a
sender with no name (`@sms1`), kept to an identifier's characters (`Alice
Martin` is addressed as `@AliceMartin`) and at most 32 of them. `people`, when
set, keeps only the names it lists. What an agent asked is recorded against
the person as the transcript labels them, and only when one person answers to
the name: a name two people answer to records nothing, and a sender who takes
another's name (labelled `Alice (2)`) never answers for them.

Only a room with a person waits: a room whose only other senders are bots and
webhooks stays idle instead, and the next event that asks for a turn opens it
again.

## Depth, turns and the end

| Option | Effect |
|--------|--------|
| `max_depth` | How far a chain of agent turns reaches from a person's message (default: the kit's `max_chain_depth`; 5 allows four hops). A turn past it is recorded once (`event_chain_depth_limit`), its agent stays queued, and the next person's message opens a new chain. |
| `max_turns` | Turns given in the room's lifetime; the discussion is over after them. |
| `done(room_id)` | Sync or async, checked before each turn: once it holds, the discussion is over. A mission's state, a person's word, a budget. |

Once over, the discussion gives no further turn and drops its queue; an
instruction addressed to an agent is refused with
`InboundResult(blocked=True, reason="discussion_over")`, and later messages
are stored and delivered with no agent asked. To have the agents answer again,
uninstall the discussion.

## Listening only

```python
await kit.listen_only(room_id, ["sre"])   # a console's "@sre hold on"
await kit.talk_again(room_id, ["sre"])    # "@sre you can carry on"
```

An agent that only listens keeps its place in the queue and takes no turn an
agent asks for; a person's message that names it gives it one turn, and it goes
on listening (a message that names nobody gives it none). Setting it on the
agent whose turn runs cuts that turn, as a `Cancel` steering directive does:
what the turn committed stays in the room; a turn given but not running yet is
abandoned before it runs. `talk_again` also ends a wait that was for that
agent.

## Instructions and regenerated answers

An `INSTRUCTION` addressed to agents queues each of them at the front for a
turn of its own that takes the instruction as its input; one addressed to no
agent of the discussion is reported in `unavailable_targets`.
`kit.regenerate_response(room_id)` queues, at the front, a turn of its own for
each agent that answered the last person's message (each agent the message
asked for, when you removed the answers first), once per agent and message.
Both are given even while the discussion waits for a person, to an agent that
only listens, and to the agent that just spoke. Past the depth limit, an
instruction's turn is dropped and reported, a regenerated answer's recorded
and dropped. Once the discussion is over, both are refused with
`discussion_over`.

## Talking and working: tasks

A turn talks; long work runs as a task, off the floor. An agent asked to go
through every support ticket used to hold the floor for the whole of it, and a
quick question to another agent waited behind. With background work set up,
it starts the work as a task, says in a sentence that it is on it, and its
turn ends: the room goes on, and the result comes back when the work is done.

```python
kit.setup_background(comms)                         # comms does its own work
kit.setup_background(dev, worker="dev-code")        # Claude Code over ACP does dev's
kit.setup_background(sre, hand_back="post")         # sre's results posted as they are
```

`setup_background` gives the agent the `work_in_background` tool. A task runs
on the agent's worker (the agent itself, or any intelligence channel the kit
holds: another agent, an external agent over ACP), in a child room, handed the
room's last 12 messages, as any delegation
([Agent Delegation](agent-delegation.md)). Tasks run in parallel with the
turns and with one another; a question asked while a task runs is answered in
its turn, not after the task.

The result comes back as the host set it up:

| `hand_back` | What happens |
|-------------|--------------|
| `"relay"` (default) | The agent is told the result and says it in a turn of its own, in the conversation's words, at the front of the queue. |
| `"post"` | The result is posted as it is, as the agent's message, with no generation: at once, and as the worker wrote it. It waits at the front of the queue for the turn under way to end, and the names it carries ask no one. |

Relay a result that needs putting in context; post one that is long or exact
(an analysis, a list, a diff), or that a slow model would hold for the length
of a turn.

A model may still do long work in its own turn whatever its tool says (in our
runs, Mistral Large 4 never used it). The host, or a person through it, starts
the task itself:

```python
task = await kit.start_task(room_id, "dev", "Read the tax engine's source: why do "
                            "Canadian checkouts fail?", hand_back="post")
await kit.cancel_task(task.id)                      # the agent says it was cancelled
```

`start_task` works for any agent of the room, an external agent included,
which takes no tool from RoomKit. Each turn's notes list the room's tasks, so
an agent asked how a task goes answers from them; `task_status` and
`cancel_task` are tools you may give it as well. A worker never starts a task
of its own: the tool is refused in a task's child room. A task outlives the
discussion: a result that comes back once it is over or uninstalled is posted
at once, or relayed as any instruction (refused once the discussion is over).

## External agents (ACP)

An external agent over ACP takes the discussion's turns as an AI channel does:

```python
from roomkit import ACPChannel

dev = ACPChannel(
    "dev",
    command=["npx", "--no-install", "@agentclientprotocol/claude-agent-acp@0.61.0"],
    cwd="/srv/checkout",
    name="Developer",
    description="reads the source code and the support tickets",
    instructions="You are @dev, a developer in an incident room. Answer briefly.",
)
room = await kit.create_room(orchestration=Discussion(agents=[investigator, dev, sre]))
```

Its prompt carries the turn's notes, under the runtime's header (a copy of
that header in what participants wrote is replaced); its `name`, `role` and
`description` present it to the others, to the dispatch policy and to the
console; `instructions` are sent once per session, since its system prompt is
not RoomKit's to set. Its session keeps its own history: a turn sends what the
agent has not read yet, then the message it answers. `listen_only` on it while
its turn runs cancels its prompt. Its tools are its own (its workspace, what
its agent accepts): the room's tools are not wired into it. A turn of a coding
agent can take tens of seconds; long work is better started with
`start_task`, which runs it in a session of its own while the room goes on.

## Following the queue

```python
from roomkit import HookExecution, HookTrigger, SpeakQueueEvent


@kit.hook(HookTrigger.ON_SPEAK_QUEUE, execution=HookExecution.ASYNC)
async def follow(event: SpeakQueueEvent, ctx) -> None:
    queue = event.queue
    print(event.change, event.channel_ids, queue.speaking, queue.queue, queue.waiting)
```

`kit.speak_queue(room_id)` returns the `SpeakQueue` at any time. It, and
`listen_only` / `talk_again`, take `organization_id` to scope the call to one
tenant: a room of another organization reads as holding no discussion.

| Field | Meaning |
|-------|---------|
| `speaking` | The agent whose turn runs now |
| `queue` | Agents owed a turn, in the order they get it |
| `listening` | Agents that only listen |
| `asked` | `(agent, person)` pairs: agents waiting for a person's answer |
| `waiting` | Whether the discussion waits for a person |
| `over` | Whether the discussion is over |

`ON_SPEAK_QUEUE` fires for each change, in the order the changes happened:
`queued`, `turn_given`, `turn_ended`, `instruction_dropped`, `listening`,
`talking_again`, `waiting` (the wait beginning or ending: read
`event.queue.waiting`), `over`.

## A console for the discussion

`DiscussionConsole` (`pip install roomkit[console]`) is a full-screen terminal
on a room a discussion holds: the room on the left (every message and tool
call, who asks you), a card per agent on the right (identity, model, live
state: speaking, next, listening, asking, nothing to add), the speak queue
below, your input at the bottom.

```python
from roomkit import WebSocketChannel
from roomkit.console import DiscussionConsole

kit.register_channel(WebSocketChannel("you"))
await kit.attach_channel(room_id, "you")
await DiscussionConsole(kit, room_id, channel_id="you", log_file="room.log").run()
```

What you type goes into the room through the transport channel you name.
`/listen @a` and `/talk @a` set the listening state, `/help` and `/quit` do
what they say, and `commands={"name": handler}` adds your own. Only messages
from your channel and sender show as yours; every other author shows under the
transcript's label. The kit's logs go to `log_file` while the screen is up
(without one, warnings are held and printed once it closes). Pass `cards` to
present the agents your way (`AgentCard`: name, role, description, model,
tools). A runnable version, with three Claude agents, is
`examples/discussion_console.py`.

With a WebSocket channel as yours, the answer an agent is writing shows as it
streams, under "writing…", until it is said. When an agent of the room has
background work set up (or with `tasks=True`), `/task @dev <what to do>`
starts a task for it, `/tasks` lists the room's tasks and `/cancel <task>`
stops one (six hex digits of its id are enough); each agent's card shows its
running tasks, and a result posted as an agent's message names the worker
that wrote it (`@dev · a task's result, by dev-code`).

## The room is the discussion's

One rule decides who speaks, never two. Installing a discussion refuses a room
that binds another intelligence channel or a voice or realtime channel, an
agent that thinks while it listens, a `ConversationRouter`, or another
strategy. While it is installed, binding such a channel, installing a router
or another strategy in the room is refused too. Transports attach as usual.
The room's `agent_response_policy` plays no part while the discussion is
installed:

```python
await kit.install_strategy(room_id, Discussion(agents=[investigator, dev, sre]))
await kit.uninstall_strategy(room_id)     # the room's policy answers again
```

A room holds one strategy at a time; it can start as a plain chat, take a
discussion when a team joins, and become a swarm later (see
[Installing a strategy while the room lives](orchestration.md#installing-a-strategy-while-the-room-lives)).

## State and processes

The discussion's configuration (`_discussion`) and speak queue (`_speak_queue`)
are stored in the room's metadata and outlive a restart: call
`strategy.install(kit, room_id)` again in the new process.

Several processes may serve one room, behind a load balancer, on one store
(`PostgresStore` with `PostgresAdvisoryLockManager`):

- every process follows the stored discussion, even one whose host did not
  install it there: it asks no agent at broadcast, reads names and queues
  turns, each change of the queue read and written under the room lock;
- one process that installed the discussion gives the turns, under a lease
  stored with the queue (15 s, renewed while held); another such process
  takes over once it expires, so a crash stalls the room 15 s at most;
- the holder reads the queue again every second, which carries another
  process's queued turn, or its `listen_only`, to it;
- an instruction's turn moves the lease to the process holding its text.

| Situation | What happens |
|-----------|--------------|
| A message reaches a worker that did not install the discussion | It is queued; the lease holder gives the turns, one at a time |
| Two messages reach two workers at once | One queue, one holder: one agent after the other |
| The worker giving the turns crashes | Its lease expires; another worker that installed the discussion takes over |

Uninstalling, in any process, forgets the discussion in all of them. A
runnable version with two simulated workers is
`examples/discussion_two_workers.py`.

## Limits of this version

- Text only: voice and realtime channels and agents that think while they
  listen are refused.
- One turn at a time: work that does not need the floor runs as tasks, but
  turns never run in parallel.
- An external agent's tools are its own: the room's tools are not wired into it.
- Text an agent writes between tool rounds reaches the room as it is written;
  only the names it carries wait for the turn's end.
- One queue per room: threads (`parent_event_id`) are not yet conversations of
  their own.

## Related guides

| Guide | Description |
|-------|-------------|
| [Multi-Agent Orchestration](orchestration.md) | The other strategies, addressing and agent response policies |
| [Speaking Turns](speaking-turns.md) | Speak policies: an agent that decides whether to answer |
| [Classifiers](classifiers.md) | Jev and LLM classifiers, the judgments a dispatch policy rests on |
| [AI Steering Directives](ai-steering.md) | `Cancel` and the other directives |
