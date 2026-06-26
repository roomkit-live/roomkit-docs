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

#### Supervised validation (sequential)

In **synchronous sequential** mode the supervisor doesn't just pass output from
one worker to the next — it acts as a reviewer between every step. For each
worker the framework:

1. Asks the supervisor to **frame** the worker's task (turning the topic, plus
   any prior validated results, into a concrete brief).
2. Runs the worker, which hands its work back as a structured result (see
   [Structured results](agent-delegation.md#structured-results)).
3. Asks the supervisor to **review** that output with a strict APPROVE / REJECT
   verdict.
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

> **Sequential only, and only when the supervisor reviews.** Validation applies
> to synchronous sequential runs (sync `auto_delegate` and strategy-tool mode).
> `async_delivery` sequential (voice/background) and `parallel` mode run workers
> without the per-step review loop.

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
5. Results delivered via `kit.deliver()` when both AI and user are idle

The `WaitForIdle` strategy waits for both the AI to finish speaking AND the user to stop talking before injecting results.

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `strategy` | `None` | `"sequential"` / `"parallel"` / `None` — how workers execute |
| `auto_delegate` | `False` | `True` = framework triggers workers automatically |
| `async_delivery` | `False` | `True` = workers run in background, results delivered via `kit.deliver()` |
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

Each channel ID listed in `share_channels` is copied from the parent room's bindings into every child room created during delegation. The child room uses the same provider instance with its own binding — events emitted through a shared channel in a child room are visible on that channel (e.g., real-time tool call status sent via a WebSocket channel).

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
| `async_delivery` | `False` | `True` = background loop, results via `kit.deliver()` |

#### Voice / real-time mode

For voice channels, `async_delivery=True` injects a `delegate_loop` tool into the RealtimeVoiceChannel. The loop runs in the background while the conversation continues:

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

The result event carries loop status in `event.metadata`:

- `approved` — `True` if all reviewers approved, `False` if max iterations reached
- `iteration` — number of iterations completed

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

## How it works

Orchestration has four layers that work together:

```
Inbound event
  -> ConversationRouter (BEFORE_BROADCAST hook, priority -100)
     -> Reads ConversationState from Room.metadata
     -> Selects agent via: affinity -> rules -> default
     -> Stamps _routed_to + _always_process on event metadata
  -> EventRouter._process_target()
     -> Checks _routed_to for INTELLIGENCE channels
     -> Skips non-targeted agents (supervisor always processes)
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
from roomkit.orchestration import ConversationState, get_conversation_state, set_conversation_state

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

# Persist (caller must save via store)
updated_room = set_conversation_state(room, new_state)
await kit.store.update_room(updated_room)
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

> [Context from previous agent (agent-triage)]: User needs help with billing. Account #12345, premium plan, last payment was 30 days ago.

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
