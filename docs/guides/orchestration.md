# Multi-Agent Orchestration

Route conversations between multiple AI agents with state tracking, rule-based routing, handoff protocol, and pipeline workflows. Agents can transfer conversations to each other while preserving context, and a supervisor can observe all exchanges.

## Quick start

```python
from roomkit import (
    AIChannel,
    ChannelCategory,
    ConversationPipeline,
    ConversationRouter,
    HandoffHandler,
    HandoffMemoryProvider,
    HookExecution,
    HookTrigger,
    PipelineStage,
    RoomKit,
    SlidingWindowMemory,
    setup_handoff,
)

# 1. Define the pipeline
pipeline = ConversationPipeline(
    stages=[
        PipelineStage(phase="intake", agent_id="agent-triage", next="handling"),
        PipelineStage(phase="handling", agent_id="agent-handler", next="review"),
        PipelineStage(
            phase="review",
            agent_id="agent-reviewer",
            next="resolution",
            can_return_to={"handling"},
        ),
        PipelineStage(phase="resolution", agent_id="agent-resolver", next=None),
    ],
    supervisor_id="agent-supervisor",
)

# 2. Generate the router
router = pipeline.to_router()

# 3. Create the framework and register channels
kit = RoomKit()
ai_triage = AIChannel("agent-triage", provider=provider, system_prompt="You triage requests.")
ai_handler = AIChannel("agent-handler", provider=provider, system_prompt="You handle requests.")
# ... register all channels

# 4. Wire handoff into each agent
handler = HandoffHandler(
    kit=kit,
    router=router,
    phase_map=pipeline.get_phase_map(),
)
for channel in [ai_triage, ai_handler]:
    channel._memory = HandoffMemoryProvider(channel._memory)
    setup_handoff(channel, handler)

# 5. Install the routing hook
kit.hook(HookTrigger.BEFORE_BROADCAST, execution=HookExecution.SYNC, priority=-100)(
    router.as_hook()
)
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
from roomkit import ConversationState, get_conversation_state, set_conversation_state

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
from roomkit import ConversationRouter, RoutingRule, RoutingConditions, ChannelType

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

### Installing the hook

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
from roomkit import HandoffHandler

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
from roomkit import setup_handoff

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
from roomkit import HandoffMemoryProvider, SlidingWindowMemory

memory = HandoffMemoryProvider(SlidingWindowMemory(max_events=50))
ai = AIChannel("agent-handler", provider=provider, memory=memory)
```

After a handoff, the receiving agent sees a prepended message like:

> [Context from previous agent (agent-triage)]: User needs help with billing. Account #12345, premium plan, last payment was 30 days ago.

## ConversationPipeline

Syntactic sugar for defining sequential multi-agent workflows. Generates a `ConversationRouter` from a list of pipeline stages.

```python
from roomkit import ConversationPipeline, PipelineStage

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

## Example

See [`examples/orchestration_pipeline.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_pipeline.py) for a runnable demo showing a multi-agent pipeline with handoff between triage, handler, and resolver agents.
