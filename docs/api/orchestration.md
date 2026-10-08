# Orchestration

Multi-agent orchestration for complex conversational workflows. See the [Multi-Agent Orchestration guide](../guides/orchestration.md) for usage examples.

## Strategies

::: roomkit.orchestration.base.Orchestration

::: roomkit.orchestration.strategies.pipeline.Pipeline

::: roomkit.orchestration.strategies.swarm.Swarm

::: roomkit.orchestration.strategies.supervisor.Supervisor

::: roomkit.orchestration.strategies.loop.Loop

## Agents

::: roomkit.Agent

::: roomkit.orchestration.ConversationRouter

::: roomkit.orchestration.RoutingRule

::: roomkit.orchestration.RoutingConditions

## Conversation State

::: roomkit.orchestration.ConversationState

::: roomkit.orchestration.ConversationPhase

::: roomkit.orchestration.PhaseTransition

::: roomkit.orchestration.get_conversation_state

::: roomkit.orchestration.set_conversation_state

::: roomkit.orchestration.save_conversation_state

## Pipeline

::: roomkit.orchestration.ConversationPipeline

::: roomkit.orchestration.PipelineStage

## Handoff

::: roomkit.orchestration.HANDOFF_TOOL

::: roomkit.orchestration.handoff.build_handoff_tool

::: roomkit.orchestration.HandoffHandler

::: roomkit.orchestration.HandoffRequest

::: roomkit.orchestration.HandoffResult

::: roomkit.orchestration.HandoffMemoryProvider

::: roomkit.orchestration.setup_handoff

## Structured results

Delegated workers can hand their result back through the `submit_result` tool
instead of a free-text message scraped from the room — a forced, parseable
handoff. See [Structured results](../guides/agent-delegation.md#structured-results)
in the Agent Delegation guide. Enabled per call via
`kit.delegate(require_structured_result=True)`, and used internally by the
supervised sequential flow.

`kit.delegate(result_tool=...)` forces another tool than `submit_result`, by
passing a `ResultTool`: the supervised flow's verdict comes back through
`submit_verdict` this way. The capture is scoped to the child room, so two
delegations to the same agent at once never read each other's result.

::: roomkit.orchestration.result.ResultTool

::: roomkit.orchestration.result.MissingResult

::: roomkit.orchestration.result.SUBMIT_RESULT

::: roomkit.orchestration.result.SUBMIT_RESULT_TOOL

::: roomkit.orchestration.result.is_submit_result

::: roomkit.orchestration.result.normalize_result

::: roomkit.orchestration.result.orchestration_fail
