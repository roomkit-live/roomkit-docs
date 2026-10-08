# Multi-Agent Orchestration

Route conversations between multiple AI agents with state tracking, rule-based routing, handoff protocol, and pipeline workflows. Agents can transfer conversations to each other while preserving context, and a supervisor can observe all exchanges.

## Quick start

The fastest way to set up multi-agent orchestration is with a **strategy** passed to `RoomKit` or `create_room`:

```python
from roomkit import Agent, Pipeline, RoomKit

triage = Agent("triage", provider=provider, role="Triage", description="Routes requests")
handler = Agent("handler", provider=provider, role="Handler", description="Resolves issues")
closer  = Agent("closer",  provider=provider, role="Closer",  description="Confirms resolution")

kit = RoomKit(orchestration=Pipeline(agents=[triage, handler, closer]))
room = await kit.create_room()
# All agents registered, attached, routing + handoff tools wired, state initialised.
```

For more control, use the lower-level primitives directly (see [ConversationPipeline](#conversationpipeline) and [ConversationRouter](#conversationrouter) below).

## Orchestration strategies

Four declarative strategies compose the existing primitives (`ConversationRouter`, `HandoffHandler`, `ConversationPipeline`) into common patterns. Pass them via `RoomKit(orchestration=...)` to apply to all rooms, or `create_room(orchestration=...)` to apply per-room.

### Pipeline

Agents are chained linearly. The first agent is the entry point; each agent can hand off to the next.

```python
from roomkit import Agent, Pipeline, RoomKit

kit = RoomKit(
    orchestration=Pipeline(
        agents=[triage, handler, resolver],
        supervisor_id="agent-supervisor",  # optional: receives all events
    ),
)
room = await kit.create_room()
```

Internally, `Pipeline` builds `PipelineStage` objects from the agent list, creates a `ConversationRouter` and `HandoffHandler`, installs room-scoped hooks, and sets the initial `ConversationState`. Each agent gets a constrained `handoff_conversation` tool whose `target` enum only includes reachable agents.

For pipelines with loops (e.g., `can_return_to`) or custom stage definitions, use `ConversationPipeline` directly — see [ConversationPipeline](#conversationpipeline).

### Swarm

Every agent can hand off to every other agent — no linear ordering. Routing relies on sticky agent affinity.

```python
from roomkit import Agent, RoomKit, Swarm

kit = RoomKit(
    orchestration=Swarm(
        agents=[sales, support, billing],
        entry="agent-sales",  # optional: defaults to first agent
    ),
)
room = await kit.create_room()
```

Each agent's `handoff_conversation` tool lists all other agents as targets. There are no phase constraints — the `HandoffHandler` allows any agent-to-agent transition.

### Supervisor

A supervisor agent talks to the user and delegates tasks to worker agents. Workers run in isolated child rooms (via `kit.delegate()`) and are NOT attached to the parent room. The framework controls the execution flow — agents only need to know their role, not how orchestration works.

> **Principle**: The agent decides the content. The framework decides the flow.

#### Framework-driven mode (`auto_delegate=True`)

The recommended mode. The framework triggers workers automatically on every user message — no tools, no AI orchestration choices:

```python
from roomkit import Agent, RoomKit, Supervisor

# Sequential: workers run in order; the supervisor validates each
# worker's output before framing the next worker's task
kit = RoomKit(
    orchestration=Supervisor(
        supervisor=coordinator,
        workers=[researcher, writer],
        strategy="sequential",
        auto_delegate=True,
    ),
)

# Parallel: both analysts run concurrently, supervisor gets combined results
kit = RoomKit(
    orchestration=Supervisor(
        supervisor=coordinator,
        workers=[technical_analyst, business_analyst],
        strategy="parallel",
        auto_delegate=True,
    ),
)
```

With `auto_delegate=True` and `refine_task=True` (default), the supervisor first
extracts a clean topic from the user's message (pass 1), then the workers run.
Pass 1 is read as any streamed turn: the workers get its final answer, not the
narration of its tool rounds, and a pass its round cap, deadline or budget cut
short hands on no task, so no worker runs. When the pass stopped short of
its answer (any end but `completed`, save `cancelled`, a stop someone chose),
the user's message still gets an answer: the supervisor's fallback ("The
delegated work could not be completed."), stored and delivered with the
turn's record (`metadata["loop_end_reason"]`, `ai_usage`, what the turn
wrote). A pass whose provider failed has no fallback: the error is the
turn's (`InboundResult.error` for a `process_inbound` caller), logged once at
its own level. The tools it calls are stored in the room as any turn's,
through `BEFORE_BROADCAST`; its text is not stored.
What happens during the worker run depends on the strategy:

**Parallel** — all workers run concurrently on the same topic, then the
supervisor presents the combined results (pass 2):

1. User sends message
2. Supervisor extracts the core topic
3. Framework runs all workers concurrently on the topic
4. Supervisor presents the combined results to the user

**Sequential** — runs as a supervised **hub-and-spoke** loop: the supervisor
frames each worker's task, reviews its output, and only then frames the next
(see [Supervised validation](#supervised-validation-sequential) below):

1. User sends message
2. Supervisor extracts the core topic
3. For each worker in order: the supervisor frames the task → the worker runs →
   the supervisor reviews the output (APPROVE/REJECT) → rejected work is sent
   back with feedback, up to `max_revisions` times → the validated result is
   carried into the next worker's brief
4. Supervisor presents the validated chain of results to the user

Agent prompts describe only **what the agent does** — no orchestration instructions needed:

```python
coordinator = Agent("coordinator", system_prompt="You coordinate analysis.")
researcher = Agent("researcher", system_prompt="You research topics thoroughly.")
writer = Agent("writer", system_prompt="You write clear articles.")
```

Pass an optional `name` (e.g. `Agent("researcher", name="Willie Researcher", ...)`)
to give an agent a human-readable display name, distinct from its `channel_id`
and `role`. Hosts use it to label a step in orchestration timelines by who ran
it instead of a truncated role sentence.

#### Supervised validation (sequential)

In **synchronous sequential** mode the supervisor doesn't just pass output from
one worker to the next — it acts as a reviewer between every step. For each
worker the framework:

1. Asks the supervisor to **frame** the worker's task (turning the topic, plus
   any prior validated results, into a concrete brief).
2. Runs the worker, which hands its work back as a structured result (see
   [Structured results](agent-delegation.md#structured-results)).
3. Asks the supervisor to **review** that output with a strict APPROVE / REJECT
   verdict, which it hands back by calling the `submit_verdict` tool (forced
   like a worker's `submit_result`, re-prompted when a turn ends without it).
   The supervisor's provider must therefore call tools. A verdict that never
   comes, or a review that times out, counts as a reject.
4. On REJECT, sends the worker the supervisor's feedback for a **rework** — up
   to `max_revisions` times.
5. Carries the validated result into the next worker's brief.

Each worker is bounded by `task_timeout` (default 120s, per worker — not a
single global budget). If a worker can't satisfy the supervisor within
`max_revisions` rounds, the step is delivered **flagged as unvalidated** rather
than looping forever — the supervisor reports an honest failure instead of
presenting unreviewed work.

```python
kit = RoomKit(
    orchestration=Supervisor(
        supervisor=editor,
        workers=[researcher, writer],
        strategy="sequential",
        auto_delegate=True,
        task_timeout=180,     # per-worker budget in seconds
        max_revisions=2,      # rework round-trips per worker
    ),
)
```

> **Sequential only.** Validation applies to every sequential run, within the
> turn (sync `auto_delegate`, strategy-tool mode) or in the background
> (`async_delivery`, voice included). `parallel` mode runs workers without the
> per-step review loop, and so does a supervisor without a model (a
> configuration-only `Agent`, as a voice supervisor often is): it cannot frame
> nor judge a step. A supervised chain that stops on a step the supervisor left
> unvalidated did not complete its work: in the background the supervisor is
> told so and the run posts `FAILED` (`a step was not validated`).

Every worker delegation, on every door (sequential, parallel, supervised,
`delegate_to_<id>` waiting or in the background), is bounded by `task_timeout`:
a worker past it is cut, its task ends `cancelled`, and it reads
`The task timed out after <n>s.` The worker is posted `PENDING` on the status
bus, then one terminal entry however its delegation ends, a delegation its
caller cancelled included (`FAILED`, detail `cancelled`).

#### Voice / real-time mode (`async_delivery=True`)

For voice and real-time channels, workers run in the background while the conversation continues. The framework injects a `delegate_workers` tool into the voice channel — the AI decides when to call it naturally:

```python
from roomkit import Agent, RoomKit, Supervisor, WaitForIdle, RealtimeVoiceChannel

kit = RoomKit(
    delivery_strategy=WaitForIdle(buffer=3.0),
    orchestration=Supervisor(
        supervisor=coordinator,
        workers=[technical, business],
        strategy="parallel",
        auto_delegate=True,
        async_delivery=True,
    ),
)
```

Flow:

1. User speaks → voice AI responds normally
2. User asks for analysis → AI calls `delegate_workers` tool
3. AI says "I'm dispatching my analysts" (natural response)
4. Workers run in background — conversation continues uninterrupted
5. Results handed back to the session that called `delegate_workers`, as an instruction, when both AI and user are idle; like a delegation's, it closes on the line that the turn it opens gives that result only, in the conversation's language

A pipeline that fails before its results (a worker delegation that raised) is handed back the same way: the supervisor is told the work could not be completed, so it can say so instead of leaving "I'll get back to you" unanswered. The error's message goes to the logs and to the status bus (`agent_id="orchestration"`, `FAILED`), never to the model. The room is free again by the time the supervisor hears the outcome, so a `delegate_workers` call it makes in answer (a retry, a follow-up) starts a new run; a supervisor that dispatches again on every outcome stops at `max_chain_depth`.

One run per room at a time, whichever voice channel's session calls `delegate_workers`: a second call while workers run answers `already_running`. With several sessions in the room, only the one that made the call is told. The run's terminal status entry (`agent_id="orchestration"`, `action="pipeline"`) is `COMPLETED` once the results are handed back, and `FAILED` when the run raised, no worker's task completed (`no worker completed`; the supervisor is then told the work could not be completed), or the outcome reached no one (its detail then says why, `not handed back: ...`).

A per-worker tool in the background (`delegate_to_<id>` with `wait_for_result=False`) is followed by the same run, for that worker in that room. Its delegation is still a task of the kit's task runner: the dispatch answer carries its `task_id`, and `kit.task_runner.cancel(task_id)` ends it. The run waits for it within `task_timeout`, frees the worker before its outcome is handed back, hands it back alone, and posts its terminal entry under `agent_id="orchestration"`, `action="worker"`.

`kit.close()` ends every background run: the run is cancelled, its worker's task ends `cancelled`, its room is freed, its terminal entry is `FAILED` (`cancelled`), and nothing is handed back.

The `WaitForIdle` strategy waits for both the AI to finish speaking AND the user to stop talking before injecting results.

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `strategy` | `None` | `"sequential"` / `"parallel"` / `None` — how workers execute |
| `auto_delegate` | `False` | `True` = framework triggers workers automatically |
| `async_delivery` | `False` | `True` = workers run in background, results handed back to the calling session |
| `refine_task` | `True` | Supervisor extracts topic before sending to workers (sync mode) |
| `refine_instruction` | `None` | Custom topic extraction instruction |
| `delegation_message` | `"I'm dispatching..."` | Message injected when workers start (async mode) |
| `wait_for_result` | `True` | Inline or background execution (manual mode) |
| `share_channels` | `None` | Channel IDs from parent room to share with worker child rooms |
| `task_timeout` | `120.0` | Per-worker delegation budget in seconds (each task bounded individually) |
| `max_revisions` | `3` | Supervised sequential: max supervisor→worker rework round-trips per step |

#### Sharing channels with workers

By default, worker child rooms only have the worker agent attached. If workers need access to channels from the parent room (e.g., a WebSocket status channel, an email channel, or a system channel for observability), use `share_channels`:

```python
kit = RoomKit(
    orchestration=Supervisor(
        supervisor=coordinator,
        workers=[researcher, writer],
        strategy="parallel",
        auto_delegate=True,
        share_channels=["system", "ws-status"],
    ),
)
```

Each channel ID listed in `share_channels` is copied from the parent room's bindings, permissions included, into every child room created during delegation. The child room uses the same provider instance with its own binding. A shared transport receives what the worker answers — its messages and tool-call rows, through the child room's `BEFORE_BROADCAST` hooks and permissions, as in any room (e.g., real-time tool call status sent via a WebSocket channel) — and never the task the supervisor sent it (RFC §23.3).

This is passed through to `kit.delegate(share_channels=...)` on every delegation call the Supervisor makes, regardless of mode (auto-delegate, strategy-based, or per-worker tools).

#### Delivery strategies

Control **when** results are delivered to the channel:

```python
from roomkit import Immediate, WaitForIdle, Queued

# Send immediately (may interrupt voice)
kit = RoomKit(delivery_strategy=Immediate())

# Wait for voice idle + buffer
kit = RoomKit(delivery_strategy=WaitForIdle(buffer=3.0))

# Batch multiple deliveries
kit = RoomKit(delivery_strategy=Queued(buffer=2.0))
```

| Strategy | Behavior |
|----------|----------|
| `Immediate()` | Deliver now, may interrupt TTS |
| `WaitForIdle(buffer)` | Wait for AI + user silence, then deliver |
| `Queued(buffer)` | Batch multiple results, deliver at next idle |

String shorthand: `strategy="wait_for_idle"`, `strategy="immediate"`, `strategy="queued"`.

#### Delivery hooks

```python
@kit.hook(HookTrigger.BEFORE_DELIVER, execution=HookExecution.ASYNC)
async def before(event, ctx):
    print(f"Delivering: {event.metadata['strategy']}")

@kit.hook(HookTrigger.AFTER_DELIVER, execution=HookExecution.ASYNC)
async def after(event, ctx):
    error = event.metadata.get("error")
    print(f"Delivered: {'failed' if error else 'ok'}")
```

#### Tool-based mode (no `auto_delegate`)

When `auto_delegate=False` (default), the AI decides when to delegate:

- With `strategy` set: a single `delegate_workers` tool is injected
- Without `strategy`: per-worker `delegate_to_<id>` tools are injected

```python
# AI decides via per-worker tools
kit = RoomKit(
    orchestration=Supervisor(
        supervisor=manager,
        workers=[researcher, coder],
    ),
)
```

### Loop

A producing agent generates output, reviewers evaluate it, and the cycle repeats until all reviewers approve or `max_iterations` is reached. The framework controls the flow — agents just produce content.

```python
from roomkit import Agent, Loop, RoomKit

# Single reviewer
kit = RoomKit(
    orchestration=Loop(
        agent=writer,
        reviewers=[editor],
        max_iterations=3,
    ),
)

# Multiple reviewers — parallel (fan-out)
kit = RoomKit(
    orchestration=Loop(
        agent=coder,
        reviewers=[security_reviewer, perf_reviewer, style_reviewer],
        strategy="parallel",
        max_iterations=3,
    ),
)

# Multiple reviewers — sequential (chained)
kit = RoomKit(
    orchestration=Loop(
        agent=coder,
        reviewers=[security_reviewer, perf_reviewer, style_reviewer],
        strategy="sequential",
        max_iterations=3,
    ),
)
```

Each iteration:

1. Producer generates content in a child room
2. Reviewers evaluate (sequential or parallel) in child rooms
3. If **all** reviewers approve (response contains "APPROVED") → loop ends
4. Otherwise → combined feedback sent back to producer for revision

Agent prompts describe only their role — no orchestration instructions:

```python
coder = Agent("coder", system_prompt="You write clean Python code.")
security = Agent("security", system_prompt="You review code for security issues.")
perf = Agent("perf", system_prompt="You review code for performance.")
```

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `agent` | — | The producing agent |
| `reviewers` | — | List of reviewing agents |
| `reviewer` | — | Single reviewer (convenience shorthand) |
| `max_iterations` | `3` | Maximum produce-review cycles |
| `strategy` | `None` | `"sequential"` / `"parallel"` for multiple reviewers |
| `async_delivery` | `False` | `True` = background loop, its outcome handed back to the voice channel that started it |

#### Voice / real-time mode

For voice channels, `async_delivery=True` injects a `delegate_loop` tool into the RealtimeVoiceChannel. The loop runs in the background while the conversation continues. Its outcome is handed back to the voice channel that started it as a background delegation's result is: an instruction to the session, the output bounded and set apart as a worker's, never a participant's message. A loop that raises hands back that the work could not be completed (the error stays in the logs and on the status bus), so the model can tell the user. The room is free for a new loop before the outcome is handed back, one loop runs per room whichever voice channel calls it, and only the session that made the call is told. The loop posts one terminal status entry (`agent_id="orchestration"`, `action="loop"`): `COMPLETED` once handed back, `FAILED` when the loop raised, its producer's task stopped it, or the outcome reached no one. `kit.close()` ends a running loop as it ends a supervisor's background run. The producer and each reviewer are posted `PENDING`, then one terminal entry however their delegation ends; the loop has no per-task bound.

```python
kit = RoomKit(
    delivery_strategy=WaitForIdle(buffer=3.0),
    orchestration=Loop(
        agent=coder,
        reviewers=[security, perf],
        strategy="parallel",
        async_delivery=True,
    ),
)
```

#### Result metadata

In the synchronous mode, the result event carries loop status in `event.metadata` (an asynchronous loop's hand-back says how it ended in its text instead):

- `approved` — `True` if all reviewers approved, `False` otherwise
- `stopped` — why the loop stopped: `approved`, `max_iterations`, or `producer_failed` (the producer's task failed after an earlier output, which is the one that went out; with no output at all the turn has no answer). Whatever stopped the loop, the caller reads how the producer's last turn ended under `InboundResult.response_metadata["turns"]["<producer>"]` (its `loop_end_reason` and `ai_usage`), as a room turn's caller does. That `ai_usage` is the producer's last turn only: the earlier iterations and the reviewers' turns are not in it, so a host that bills from `turns` reads each delegated task's own result (`DelegatedTaskResult.metadata["turns"]`) instead. On `producer_failed`, a cut (the producer's round cap, deadline, budget, a stop) is read there and is no error, firing no `ON_ERROR`, as a room turn's cut does; a turn that failed reaches the caller on `InboundResult.error` as the error it raised, its type unchanged (a `ProviderError` stays one), whether an output went out or not, and `ON_ERROR` fires once, from the producer's own turn in its task's room
- `iteration` — number of iterations completed: the one whose output went out

```python
@kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC)
async def on_loop_result(event, ctx):
    if "approved" not in (event.metadata or {}):
        return
    if event.metadata["approved"]:
        print(f"Approved after {event.metadata['iteration']} iteration(s)")
    else:
        print(f"Not approved after {event.metadata['iteration']} iteration(s)")
```

### Per-room override

The kit-level default can be overridden (or disabled) per room:

```python
kit = RoomKit(orchestration=Pipeline(agents=[a, b]))

# Uses kit default
room1 = await kit.create_room()

# Overrides with a different strategy
room2 = await kit.create_room(orchestration=Swarm(agents=[x, y, z]))

# Disables orchestration for this room
room3 = await kit.create_room(orchestration=None)
```

A strategy is installed per room, but its agents (and a voice channel it
serves) serve every room they are attached to. What it adds for one room is
set up for that room on the shared agent or channel, with that install's
configuration, and reaches that room only (RFC §19.7):

- its tools are declared in the room's turns and in the room's realtime
  sessions: a supervisor's `delegate_workers` or `delegate_to_<worker>`, a
  voice supervisor's `delegate_workers`, a voice loop's `delegate_loop`, the
  handoff tool Pipeline and Swarm set up on each agent, a delegation's
  `submit_result` in its child room. Not in the supervisor's other rooms, nor
  in the `::task-` rooms where it frames and judges its workers' steps;
- each runs with the install of the room it is called from: two rooms that
  install a `Supervisor` on the same agent with different workers each run
  their own, whichever was installed first;
- a sync `Loop` or an auto-delegating `Supervisor` takes the turns of the
  rooms it was installed in; in a room where it was not, the agent answers as
  itself.

Nothing is written onto the shared agent or channel for one room: no wrapped
handler, no rewritten prompt, no default another room would read. A busy
worker or a running pipeline blocks its own room's next call, never another
room's. `setup_handoff(agent, handler)` without a room sets the handoff up for
every room the agent serves; `setup_handoff(..., room_id=...)` for one.

The installs are in memory: after a restart, install the strategy again in the
rooms that already exist (`strategy.install(kit, room_id)`), as
`create_room(orchestration=...)` does for a new one.

### Custom strategies

Subclass `Orchestration` to build your own:

```python
from roomkit.orchestration.base import Orchestration

class MyStrategy(Orchestration):
    def agents(self) -> list[Agent]:
        """Agents to register and attach to the room."""
        return [...]

    async def install(self, kit: RoomKit, room_id: str) -> None:
        """Wire hooks, tools, and state into the room."""
        ...
```

## Addressing: naming who is asked

Routing rules answer *which agent handles this kind of event*. Addressing
answers the question a human asks every time they type: **which agent am I
talking to right now**. A message can name its recipients:

```python
await kit.process_inbound(
    InboundMessage(
        channel_id="you",
        sender_id="user",
        content=TextContent(body="review hello.py"),
        addressed_to=["codex"],      # only this agent is asked to act
    )
)
```

**An address is not visibility.** `visibility` is configured on a binding and
says who may *see* what a source produces; an address rides on the event and
says who is *asked to act*. Addressing one agent hides the event from nobody,
and transport delivery is never narrowed — the humans in the room still get
the message.

| `addressed_to` | Meaning |
|---|---|
| `None` | Unaddressed — every eligible agent acts, or the router decides |
| `["codex"]` | Only `codex` is asked; the others see it and stay silent |
| `[]` | Nobody is asked — a decision, not an absence |

The address is stored on the event, so a transcript can show who was asked and
a replay reproduces the same solicitation.

Direct injection has a sender too, so `send_event()` addresses the same way.
The `[]` case is what an application needs when it stores a message and
triggers the answer itself: the stored event must not *also* wake the agent.

```python
# Stored, and asking nobody — the caller runs the turn on its own terms.
await kit.send_event(
    room_id=room_id,
    channel_id="system",
    content=TextContent(body=body),
    addressed_to=[],
)
```

**RoomKit takes the decision, never the syntax.** `@codex`, `/agent codex`, a
keyboard picker, a Slack payload carrying its own mentions — how a user names
an agent is your application's business. Parse it at the edge, pass channel
ids.

### Directing an agent: instructions

Sometimes the application, not a participant, needs an agent to speak: a
handoff asks the new agent to introduce itself, a schedule nudges it. Sent as
an ordinary message, that direction is stored as someone's words, shown in the
transcript as theirs, and read by the model as a user turn. Send it as an
`INSTRUCTION` instead:

```python
await kit.process_inbound(
    InboundMessage(
        channel_id="voice",
        sender_id="system",
        event_type=EventType.INSTRUCTION,
        content=TextContent(body="Handoff complete. Introduce yourself to the caller."),
        addressed_to=["advisor"],   # required: which agent is directed
    ),
    room_id=room_id,
)
# The room now holds one message: the advisor's own introduction.
```

It goes through the same pipeline — hooks, the room's order, addressing — and:

- **is never stored**, blocked or not, and consumes no index;
- **reaches only the agents it addresses** (an unaddressed one is refused,
  `reason="instruction_unaddressed"`), never a transport, so no voice channel
  speaks it;
- **is the agent's input for one turn**, marked as the application's, never as
  a participant's line, and absent from later turns' history;
- **is recorded on the reply** as a fingerprint:
  `metadata["instruction"] == {"sha256": "…", "length": 51}` (length in code
  points) says an
  instruction made the agent speak, and which one, without copying its text
  onto every reply (an application that shows the text keeps it and matches it
  by the digest).

A pass that must start from a blank page, such as a summary re-run that would
otherwise read and copy its previous answer, sends a **standalone**
instruction: its turn reads nothing of the room. The input is the instruction
alone, and the agent's memory provider is not called at all, so a provider
that always keeps a minimum of events, or returns a summary, adds nothing
either. Nor does the turn see the room's working memories: skills activated in
the room, its plan, or the digest of tools used there (which quotes their
results). The channel's own system prompt, tools and skill catalogue are
unchanged.

```python
await kit.process_inbound(
    InboundMessage(
        channel_id="voice",
        sender_id="system",
        event_type=EventType.INSTRUCTION,
        content=TextContent(body=summary_prompt),
        addressed_to=["summarizer"],
        standalone=True,   # no history, no memory provider call
    ),
    room_id=room_id,
)
```

`send_event(..., standalone=True)` does the same. `standalone` on any other
event type is refused: on a message, it would store a participant's line
without the isolation it asked for.

It takes no `idempotency_key` (refused, `instruction_not_idempotent`): there is
no stored event to find it again by. A realtime session is directed with
`inject_text(role="system")` instead — see
[What `inject_text`'s role means](realtime-voice-providers.md#what-inject_texts-role-means).

### Do agents answer each other?

An agent's output is an event like any other, so by default it solicits the
other agents in the room — the chaining that makes a pipeline work
(analyst → writer), bounded by `max_chain_depth`. In a room of *independent*
agents that is a hazard: two agents answer each other until the depth limit
stops them. The room says which it wants:

```python
kit = RoomKit(agent_response_policy=AgentResponsePolicy.ADDRESSED_ONLY)
await kit.create_room(room_id="dev", agent_response_policy=...)  # per room
```

| Policy | An agent's output solicits |
|---|---|
| `AGENT_CHAIN` | every eligible intelligence channel — the default |
| `ADDRESSED_ONLY` | only the channels it addressed, if any |

It lives on the `Room` and is persisted, not passed at construction: a policy
consulted on every broadcast must reach the same verdict in whichever worker
owns the delivery lane. A room created before the setting existed reads
`agent_chain`, the behaviour it was already running under. Under either policy
an explicit address is honoured — an agent may address another agent and be
answered by it alone.

### A room that becomes multi-agent

A room rarely knows at creation how many agents it will end up holding. A chat
that starts with one assistant and gains a second when the user asks for it is
a *different room* the moment that second one attaches: under `AGENT_CHAIN` the
first answer solicits the newcomer, whose answer comes back, down to
`max_chain_depth`. Switch the live room at that moment:

```python
await kit.attach_channel(room_id, "codex", category=ChannelCategory.INTELLIGENCE)
await kit.set_agent_response_policy(room_id, AgentResponsePolicy.ADDRESSED_ONLY)
```

The change applies to events processed after it; an event already broadcast is
not reconsidered. Setting the policy a room already holds is a no-op, so the
call sits safely on the attach path.

Growing the roster this way costs nothing at rest. A binding whose channel is
not registered — the state after a restart, before the roster is rehydrated —
is skipped silently when the event does not address it, so only the agent
actually being talked to has to be rebuilt.

An address **outranks the router**: a `ConversationRouter` returns untouched
on an addressed event and stamps nothing. Without that precedence the two
mechanisms could not coexist — sticky affinity is consulted before the rules,
so a rule written to honour an address would be unreachable for as long as any
agent held the conversation.

## How it works

Orchestration has four layers that work together:

```
Inbound event
  -> ConversationRouter (BEFORE_BROADCAST hook, priority -100)
     -> Addressed event? -> returns untouched, the address decides
     -> Reads ConversationState from Room.metadata
     -> Selects agent via: affinity -> rules -> default
     -> Stamps _routed_to + _always_process on event metadata
  -> EventRouter._process_target()
     -> Checks the address first, then _routed_to, for INTELLIGENCE channels
     -> Skips non-solicited agents (supervisor always processes)
  -> Active agent generates response
     -> May call handoff_conversation tool
  -> HandoffHandler processes handoff
     -> Updates ConversationState
     -> Persists to Room.metadata
     -> Emits system event
     -> Next inbound routes to new agent
```

## ConversationState

Tracks conversation progress within a room. Stored in `Room.metadata["_conversation_state"]` and round-trips through JSON cleanly.

```python
from roomkit.orchestration import ConversationState, get_conversation_state, save_conversation_state

# Read state from a room
state = get_conversation_state(room)
print(state.phase)            # "intake"
print(state.active_agent_id)  # "agent-triage" or None
print(state.handoff_count)    # 0

# Transition to a new phase (immutable — returns a new instance)
new_state = state.transition(
    to_phase="handling",
    to_agent="agent-handler",
    reason="User request classified as billing issue",
)

# Persist its metadata key alone: a full room write from the room read
# above would undo what was written since
await save_conversation_state(kit.store, room.id, new_state)
```

### ConversationPhase

Six built-in phase names are provided as a `StrEnum`. You can use any string as a phase name — routing and state do not restrict phases to this enum.

| Phase | Value |
|-------|-------|
| `INTAKE` | `"intake"` |
| `QUALIFICATION` | `"qualification"` |
| `HANDLING` | `"handling"` |
| `ESCALATION` | `"escalation"` |
| `RESOLUTION` | `"resolution"` |
| `FOLLOWUP` | `"followup"` |

### PhaseTransition

Every call to `state.transition()` appends a `PhaseTransition` audit record to `state.phase_history`:

```python
for t in state.phase_history:
    print(f"{t.from_phase} -> {t.to_phase} by {t.from_agent} ({t.reason})")
```

## ConversationRouter

Routes events to agents using a three-tier selection strategy:

1. **Agent affinity** — If `active_agent_id` is set and the agent is still in the room, it keeps handling
2. **Rule matching** — Evaluate `RoutingRule` objects in priority order; first match wins
3. **Default fallback** — Fall back to `default_agent_id`

Events from intelligence channels are never routed (prevents loops).

### RoutingRule and RoutingConditions

```python
from roomkit import ChannelType
from roomkit.orchestration import ConversationRouter, RoutingRule, RoutingConditions

router = ConversationRouter(
    rules=[
        RoutingRule(
            agent_id="agent-billing",
            conditions=RoutingConditions(
                phases={"handling"},
                intents={"billing"},
            ),
            priority=0,
        ),
        RoutingRule(
            agent_id="agent-support",
            conditions=RoutingConditions(
                phases={"handling"},
                channel_types={ChannelType.SMS},
            ),
            priority=10,
        ),
    ],
    default_agent_id="agent-triage",
    supervisor_id="agent-supervisor",
)
```

All conditions within a rule are ANDed. Available condition fields:

| Field | Type | Description |
|-------|------|-------------|
| `phases` | `set[str]` | Match when conversation is in one of these phases |
| `channel_types` | `set[ChannelType]` | Match when event source is one of these channel types |
| `intents` | `set[str]` | Match when `event.metadata["intent"]` is in this set |
| `source_channel_ids` | `set[str]` | Match when event comes from one of these channels |
| `custom` | `Callable` | Custom predicate `(event, context, state) -> bool` |

### Supervisor

The `supervisor_id` agent always receives events regardless of routing. Use this for oversight, logging, or intervention.

### One-liner setup

Use `router.install()` to register the hook and wire handoff on all agents in one call:

```python
handler = router.install(
    kit,
    [ai_triage, ai_billing, ai_tech],
    agent_aliases={"billing": "agent-billing"},
    phase_map={"agent-billing": "handling"},
)
```

### Manual setup

For full control, register the hook and handoff separately:

```python
kit.hook(HookTrigger.BEFORE_BROADCAST, execution=HookExecution.SYNC, priority=-100)(
    router.as_hook()
)
```

The hook runs at priority `-100` (before user hooks) and stamps `_routed_to` and `_always_process` on event metadata. The `EventRouter` reads these fields to filter intelligence channels.

## Handoff Protocol

Agents trigger handoffs by calling the `handoff_conversation` tool. The framework intercepts the call, validates the target, updates state, and emits a system event.

### HandoffHandler

```python
from roomkit.orchestration import HandoffHandler

handler = HandoffHandler(
    kit=kit,
    router=router,
    agent_aliases={"billing": "agent-billing", "human": "human"},
    phase_map={"agent-billing": "handling", "agent-resolver": "resolution"},
    allowed_transitions=pipeline.get_allowed_transitions(),  # enforce pipeline topology
)
```

| Parameter | Description |
|-----------|-------------|
| `kit` | The `RoomKit` instance (for room access and event emission) |
| `router` | The `ConversationRouter` (for rule validation) |
| `agent_aliases` | Map friendly names to channel IDs (e.g., `"billing"` -> `"agent-billing"`) |
| `phase_map` | Map agent IDs to default phases (used when `next_phase` not specified) |
| `allowed_transitions` | Optional `dict[str, set[str]]` from `pipeline.get_allowed_transitions()`. When set, handoffs to disallowed phases are rejected. |

### setup_handoff

Wires the handoff tool into an AIChannel:

```python
from roomkit.orchestration import setup_handoff

setup_handoff(ai_channel, handler)
```

This does two things:

1. Injects `HANDOFF_TOOL` into the channel's tool definitions
2. Wraps the tool handler to intercept `handoff_conversation` calls

A handoff acts on the room of the call that requested it, read from the tool
call context (`current_tool_room_id()`), with or without a `ConversationRouter`.
The same holds for every orchestration tool: a supervisor's
`delegate_to_<worker>` and `delegate_workers`, and `delegate_task`. One agent
attached to several rooms therefore hands off, or delegates, from each room on
its own, and a child room's parent is the room whose call asked for it. Called
directly, outside a tool call, these tools have no room to act on and refuse:
to demonstrate one without a model, script the call through a
`MockAIProvider(ai_responses=[...])`, as the mock examples below do.

The handoff tool definition tells the AI when and how to transfer:

```json
{
  "name": "handoff_conversation",
  "parameters": {
    "required": ["target", "reason", "summary"],
    "properties": {
      "target": "Target agent ID or alias",
      "reason": "Why the handoff is needed",
      "summary": "Context for the next agent",
      "next_phase": "Optional phase to transition to",
      "channel_escalation": "same | voice | email | sms"
    }
  }
}
```

### Human escalation

The special target `"human"` sets `active_agent_id` to `None`, allowing all agents to process events (or none, depending on your rules). This is the escape hatch for human-in-the-loop workflows.

### HandoffMemoryProvider

Wraps an inner `MemoryProvider` to inject handoff context when a conversation has been transferred:

```python
from roomkit.orchestration import HandoffMemoryProvider
from roomkit.memory import SlidingWindowMemory

memory = HandoffMemoryProvider(SlidingWindowMemory(max_events=50))
ai = AIChannel("agent-handler", provider=provider, memory=memory)
```

After a handoff, the receiving agent sees a prepended message like:

```text
[Context from previous agent (agent-triage)]
<conversation_summary>
User needs help with billing. Account #12345, premium plan, last payment was 30 days ago.
</conversation_summary>
```

The summary is the previous agent's output, set apart in a block it cannot
close (RFC §6.4). The same holds for everything a strategy hands a model from
another model: each worker's output, a reviewer's feedback and the content a
reviewer judges are a `<worker_output>` block of their own under their label,
and the user's goal copied into such an input is a `<task>` block. The task a
supervisor frames for a worker stays that worker's own input.

## ConversationPipeline

Syntactic sugar for defining sequential multi-agent workflows. Generates a `ConversationRouter` from a list of pipeline stages.

```python
from roomkit.orchestration import ConversationPipeline, PipelineStage

pipeline = ConversationPipeline(
    stages=[
        PipelineStage(phase="analysis", agent_id="agent-discuss", next="coding"),
        PipelineStage(phase="coding", agent_id="agent-coder", next="review"),
        PipelineStage(
            phase="review",
            agent_id="agent-reviewer",
            next="report",
            can_return_to={"coding"},  # Reviewer can send back to coder
        ),
        PipelineStage(phase="report", agent_id="agent-writer", next=None),
    ],
    supervisor_id="agent-supervisor",
)

router = pipeline.to_router()
```

### One-liner setup

Use `pipeline.install()` to generate the router, register the hook, and wire handoff on all agents:

```python
router, handler = pipeline.install(kit, [ai_triage, ai_handler, ai_reviewer])
```

You can pass `agent_aliases` and `hook_priority` as keyword arguments. The handler is created with `phase_map` and `allowed_transitions` derived from the pipeline stages automatically.

### PipelineStage fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | `str` | Phase name for this stage |
| `agent_id` | `str` | Agent channel ID that handles this phase |
| `next` | `str \| None` | Phase to transition to after this stage |
| `can_return_to` | `set[str]` | Additional phases this stage can transition back to |

### Pipeline utilities

```python
# agent_id -> default phase mapping (for HandoffHandler.phase_map)
pipeline.get_phase_map()
# {"agent-discuss": "analysis", "agent-coder": "coding", ...}

# phase -> allowed next phases (for validation)
pipeline.get_allowed_transitions()
# {"analysis": {"coding"}, "coding": {"review"}, "review": {"report", "coding"}, ...}
```

### Validation

The pipeline validates its graph at construction:

- `next` must reference an existing phase
- `can_return_to` entries must reference existing phases
- Self-referencing (`next="self_phase"`) is allowed for loops

## Hook triggers

Three orchestration-specific hook triggers are available:

| Trigger | Description |
|---------|-------------|
| `ON_PHASE_TRANSITION` | Fired when the conversation phase changes |
| `ON_HANDOFF` | Fired when a handoff is accepted |
| `ON_HANDOFF_REJECTED` | Fired when a handoff is rejected (target not found) |

## Related guides

| Guide | Description |
|-------|-------------|
| [Delivery Service](delivery.md) | `kit.deliver()` with WaitForIdle, Immediate, Queued strategies |
| [Agent Delegation](agent-delegation.md) | Delegate tasks to background agents |
| [Status Bus](status-bus.md) | Share real-time status between agents |
| [Tool Auditing](tool-audit.md) | Record and inspect tool calls |
| [Telemetry](telemetry.md) | Span and metric collection |

## Examples

### Strategies

| Example | Description |
|---------|-------------|
| [`orchestration_pipeline_cli.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_pipeline_cli.py) | Pipeline: triage → handler → resolver (CLI + Anthropic) |
| [`orchestration_swarm_cli.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_swarm_cli.py) | Swarm: sales ↔ support ↔ billing (CLI + Anthropic) |
| [`orchestration_supervisor_sequential_content_workflow.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_supervisor_sequential_content_workflow.py) | Supervisor: researcher → writer sequential (CLI) |
| [`orchestration_supervisor_parallel_tasks.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_supervisor_parallel_tasks.py) | Supervisor: technical + business parallel (CLI) |
| [`orchestration_supervisor_voice_parallel.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_supervisor_voice_parallel.py) | Supervisor: parallel + async_delivery (Grok voice) |
| [`orchestration_loop_cli.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_loop_cli.py) | Loop: writer + 3 parallel reviewers (CLI) |
| [`orchestration_approval_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_approval_loop.py) | Loop: mock produce/review cycle |

### Mock examples (no API key needed)

| Example | Description |
|---------|-------------|
| [`orchestration_pipeline.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_pipeline.py) | Pipeline with MockAIProvider |
| [`orchestration_swarm.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_swarm.py) | Swarm with MockAIProvider |
| [`orchestration_supervisor.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_supervisor.py) | Supervisor manual mode with MockAIProvider |

### Advanced

| Example | Description |
|---------|-------------|
| [`orchestration_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_loop.py) | ConversationPipeline with `can_return_to` loops |
| [`orchestration_routing.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_routing.py) | ConversationRouter with custom rules and supervisor |
| [`orchestration_voice_triage.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_voice_triage.py) | Voice call with delegation to background agent |
