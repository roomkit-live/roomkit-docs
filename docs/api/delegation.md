# Delegation

Background task delegation via child rooms. See the [Agent Delegation guide](../guides/agent-delegation.md) for usage examples.

## Task Runner

::: roomkit.tasks.TaskRunner

::: roomkit.tasks.InMemoryTaskRunner

## Task Models

::: roomkit.tasks.DelegatedTask

::: roomkit.tasks.DelegatedTaskResult

## Tool Integration

::: roomkit.tasks.DELEGATE_TOOL

::: roomkit.tasks.build_delegate_tool

::: roomkit.tasks.DelegateHandler

::: roomkit.tasks.setup_delegation

::: roomkit.tasks.setup_realtime_delegation

## Background Work

`RoomKit.setup_background(agent, worker=None, hand_back="relay")` gives an
agent the `work_in_background` tool (`BACKGROUND_TOOL`, from `roomkit.tasks`),
and `RoomKit.start_task(room_id, agent_id, task, hand_back=None)` starts the
same task from the host. See [Long work from an
agent](../guides/agent-delegation.md#long-work-from-an-agent-work_in_background).

::: roomkit.tasks.BackgroundSetup

::: roomkit.core.mixins.BackgroundMixin

## Following Tasks

::: roomkit.tasks.TaskStatusTool

::: roomkit.tasks.CancelTaskTool

::: roomkit.tasks.status.room_tasks

::: roomkit.tasks.post_task_progress

The tools' names, as a model calls them: `TASK_STATUS_TOOL` (`"task_status"`) and
`CANCEL_TASK_TOOL` (`"cancel_task"`), from `roomkit.tasks`.

## Delegation State Tracking

::: roomkit.tasks.CompletedTaskCache

## Delivery Strategies

::: roomkit.DeliveryStrategy

::: roomkit.Immediate

::: roomkit.WaitForIdle

::: roomkit.Queued

::: roomkit.DeliveryContext
