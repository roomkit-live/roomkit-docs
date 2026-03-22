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

# Sequential: researcher runs first, writer receives researcher's output
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

With `auto_delegate=True` and `refine_task=True` (default), the flow is:

1. User sends message
2. Supervisor extracts the core topic (pass 1 — framework-injected instruction)
3. Framework runs workers with the topic (sequential or parallel)
4. Supervisor presents combined results to user (pass 2)

Agent prompts describe only **what the agent does** — no orchestration instructions needed:

```python
coordinator = Agent("coordinator", system_prompt="You coordinate analysis.")
researcher = Agent("researcher", system_prompt="You research topics thoroughly.")
writer = Agent("writer", system_prompt="You write clear articles.")
```

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

A producing agent generates output, a reviewer evaluates it, and the cycle repeats until the reviewer calls `approve_output` or `max_iterations` is reached.

```python
from roomkit import Agent, Loop, RoomKit

kit = RoomKit(
    orchestration=Loop(
        agent=writer,
        reviewer=editor,
        max_iterations=3,
    ),
)
room = await kit.create_room()
```

The reviewer gets an `approve_output` tool that sets `_loop_approved` in room metadata. An `AFTER_BROADCAST` hook drives the cycle: producer output → reviewer, reviewer feedback → producer. Loop state (`_loop_iteration`, `_loop_approved`) is tracked in `ConversationState.context`.

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
| [Agent Delegation](agent-delegation.md) | Delegate tasks to background agents |
| [Status Bus](status-bus.md) | Share real-time status between agents |
| [Tool Auditing](tool-audit.md) | Record and inspect tool calls |
| [Telemetry](telemetry.md) | Span and metric collection |

## Examples

### Strategies

| Example | Description |
|---------|-------------|
| [`orchestration_pipeline.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_pipeline.py) | Pipeline strategy: linear agent chain with handoff |
| [`orchestration_swarm.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_swarm.py) | Swarm strategy: bidirectional handoff between agents |
| [`orchestration_supervisor.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_supervisor.py) | Supervisor strategy: delegation to workers in child rooms |
| [`orchestration_approval_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_approval_loop.py) | Loop strategy: producer/reviewer approval cycle |

### Advanced (lower-level primitives)

| Example | Description |
|---------|-------------|
| [`orchestration_loop.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_loop.py) | ConversationPipeline with `can_return_to` loops |
| [`orchestration_routing.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_routing.py) | ConversationRouter with custom rules and supervisor |
| [`orchestration_voice_triage.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_voice_triage.py) | Voice call with delegation to background agent |
| [`screen_agent_orchestrated.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/screen_agent_orchestrated.py) | Voice + exec agent with OmniView vision |
