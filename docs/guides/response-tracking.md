# Response Tracking

An agent can have several answers under way in one room at once: the answer to
a question, a background task's result coming back, an offer of its own.
RoomKit does not hold the room while a model generates, so these answers run
side by side. To keep them apart, **every answer names the request it
answers** (RFC §8.5).

```
a1 « donne-moi la météo »          → r1 answers a1   (responds_to = a1)
b1 « il en est où le compteur ? »  → r2 answers b1   (responds_to = b1)
```

With that link a host can tell each turn what it answers, and see what the
room's other requests are getting. That keeps one answer from also covering a
request another answer is already handling.

## What carries the link

`RoomEvent.responds_to` is set on **every event an intelligence channel
produces for a turn**, and holds the id of the event that triggered the turn:

| Produced for the turn | `responds_to` |
|---|---|
| The answer's messages and streamed text segments | the trigger's id |
| Its tool rows (`TOOL_CALL_START` / `TOOL_CALL_END`) | the trigger's id |
| A blocked stand-in for an answer not given (chain-depth limit) | the trigger's id |
| The `ON_ERROR` event of a turn that failed | the trigger's id |
| An inbound message, a system event, a greeting | `None` |

The trigger is whatever started the turn:

- **A participant's message.** Its id is in the timeline.
- **An instruction**, sent with `kit.deliver(..., instruction=True)`. A
  delegated task's hand-back arrives as one. An instruction is never stored,
  but it has an id, which its `BEFORE_BROADCAST` hooks see. The answer names
  that id even though the timeline has no row for it.
- **An agent's message**, when one agent answers another.

The pair (`source.channel_id`, `responds_to`) names one agent's answer to one
request.

`responds_to` is not the thread: `parent_event_id` places an event in an in-app
thread and always names the thread's root (see
[Message Threading](message-threading.md)). An answer in a thread carries both.

## Telling the turn what it answers

`BEFORE_AI_GENERATION` receives the trigger as `event.trigger`:

```python
from roomkit import HookResult, HookTrigger, add_turn_note

@kit.hook(HookTrigger.BEFORE_AI_GENERATION)
async def scope_the_turn(event, ctx) -> HookResult:
    trigger = event.trigger
    note = f"This turn answers: {trigger.content.body!r}. Answer that only."
    event.ai_context.messages = add_turn_note(event.ai_context.messages, note)
    return HookResult.allow()
```

An orchestration strategy can hand a turn a stand-in in place of the event it
answers, such as a supervisor's workers' results. The stand-in names the event
it stands for in its own `responds_to`, and the turn's answer names that same
event.

## Finding the answers to a request

```python
from roomkit.models.store_filter import EventFilter

answers = await kit.store.list_events(
    "room-1", event_filter=EventFilter(responds_to=question_id)
)
```

Every store supports the filter. SQLite and Postgres keep `responds_to` in an
indexed column.

## Your own channels

A channel that returns buffered answers (`ChannelOutput.response_events`) does
not need to set the field: RoomKit fills in the trigger's id. A channel that
already sets `responds_to` keeps its value. That is how an answer can name an
earlier event than the one that woke it.

## Upgrading stores

- **SQLite.** Schema version 4 adds the `responds_to` column and its index. A
  file at version 1, 2 or 3 is migrated when it is opened. Events written
  before the upgrade read `None`.
- **Postgres.** `init()` adds the column and a partial index
  (`ALTER TABLE ... ADD COLUMN IF NOT EXISTS`). Existing rows are `NULL`, which
  is what they were.

## Not covered yet

Speech-to-speech channels (realtime voice, conference) answer the user's speech
before that speech's transcript is stored, so there is no event to name yet.
Their answers carry `responds_to = None` (RFC §8.5, *Planned*).

Whether an answer was actually **taken in** (read, heard to the end, cut, or
never played), per channel and per participant, comes with the second part of
the same RFC section: response consumption.

## Example

`examples/response_tracking.py` runs a question and an instruction through an
assistant and prints what each turn answers:

```
turn of assistant answers 270acc92 (message): 'Quelle heure est-il ?'
turn of assistant answers ff0880d1 (instruction): 'Share the weather result: 8 degrees tomorrow.'
  #2 ws-user: 'Quelle heure est-il ?'
  #3 assistant: 'Il est midi.' → answers 270acc92
  #4 assistant: 'La météo est arrivée : 8 degrés demain.' → answers ff0880d1
Answers to 270acc92: ['Il est midi.']
```
