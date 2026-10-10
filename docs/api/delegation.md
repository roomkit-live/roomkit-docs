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

## Background Work

`RoomKit.setup_background(agent, worker=None, hand_back="relay")` gives an
agent the `work_in_background` tool (`BACKGROUND_TOOL`, from `roomkit.tasks`),
and `RoomKit.start_task(room_id, agent_id, task, hand_back=None)` starts the
same task from the host. See [Long work from an
agent](../guides/agent-delegation.md#long-work-from-an-agent-work_in_background).

::: roomkit.tasks.background.BackgroundSetup

::: roomkit.core.mixins.background.BackgroundMixin

## Following Tasks

::: roomkit.tasks.status.TaskStatusTool

::: roomkit.tasks.status.CancelTaskTool

::: roomkit.tasks.status.room_tasks

::: roomkit.tasks.status.post_task_progress

The tools' names, as a model calls them: `TASK_STATUS_TOOL` (`"task_status"`) and
`CANCEL_TASK_TOOL` (`"cancel_task"`), from `roomkit.tasks`.

## Delegation State Tracking

::: roomkit.tasks.cache.CompletedTaskCache

## Delivery Strategies

::: roomkit.core.delivery.DeliveryStrategy

::: roomkit.core.delivery.Immediate

::: roomkit.core.delivery.WaitForIdle

::: roomkit.core.delivery.Queued

::: roomkit.core.delivery.DeliveryContext
