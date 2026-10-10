# Discussion: Agents in a Group Chat

Several agents and one or more people hold one conversation in a room. Any
agent may address any other with `@channel_id`, a person may address any
agent, and nobody fixes the order in advance: who speaks next follows from who
was addressed. One agent speaks at a time (RFC §19.7.5).

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
| A person's message naming nobody | The agents that asked that person, else `everyone`, in its order |
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
`addressed_only` off and give them a [speak policy](speaking-turns.md).

## What a turn reads

Each turn is a rerun of the event it answers, planned for one agent and run in
the room's delivery lane. Its context is built when the turn starts, so what
was said while the agent waited is in it, and the event it answers keeps its
place in the history instead of being read as the last message. The turn's
notes tell the agent who else is in the room (their channel ids, with the
`name`, `role` and `description` of Agents), whom it may address, who asked
for this turn, and how the room works.

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

- Text only: voice and realtime channels, agents that think while they listen
  and external agents (ACP) are refused.
- One turn at a time, even for agents whose work does not depend on each other.
- Text an agent writes between tool rounds reaches the room as it is written;
  only the names it carries wait for the turn's end.
- One queue per room: threads (`parent_event_id`) are not yet conversations of
  their own.

## Related guides

| Guide | Description |
|-------|-------------|
| [Multi-Agent Orchestration](orchestration.md) | The other strategies, addressing and agent response policies |
| [Speaking Turns](speaking-turns.md) | Speak policies: an agent that decides whether to answer |
| [AI Steering Directives](ai-steering.md) | `Cancel` and the other directives |
