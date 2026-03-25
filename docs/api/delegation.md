# Delegation

Background task delegation via child rooms. See the [Agent Delegation guide](../guides/agent-delegation.md) for usage examples.

## Task Runner

::: roomkit.tasks.base.TaskRunner

::: roomkit.tasks.memory.InMemoryTaskRunner

## Task Models

::: roomkit.tasks.models.DelegatedTask

::: roomkit.tasks.models.DelegatedTaskResult

## Tool Integration

::: roomkit.tasks.delegate.DELEGATE_TOOL

::: roomkit.tasks.delegate.build_delegate_tool

::: roomkit.tasks.delegate.DelegateHandler

::: roomkit.tasks.delegate.setup_delegation

::: roomkit.tasks.delegate.setup_realtime_delegation

## Delegation State Tracking

::: roomkit.tasks.cache.CompletedTaskCache

## Delivery Strategies

::: roomkit.core.delivery.DeliveryStrategy

::: roomkit.core.delivery.Immediate

::: roomkit.core.delivery.WaitForIdle

::: roomkit.core.delivery.Queued

::: roomkit.core.delivery.DeliveryContext
