# Multi-Speaker Rooms

`AIChannel` builds the model's history from the room's events: every event that is not the AI's own becomes a `user` turn. In a 1:1 conversation that is exactly right. In a room where several people speak — a Teams channel, a WhatsApp group, a shared inbox with two agents on it — it erases who said what. The model sees one anonymous stream, guesses the addressee from the prompt's single audience line, and guesses wrong: a reply that opens with the wrong colleague's name.

Since roomkit 0.59.0 the AI channel attributes user turns to their speaker whenever the history window holds **two or more distinct speakers**: a person by name, or a channel whose sender has no name (the runtime's system events aside). There is nothing to configure; this guide explains what the model receives, where the speaker comes from, and how to make sure your events carry one.

## What the model receives

Three people in a room, the AI replying to the last message:

```text
system:    You are the team's scheduling assistant.

           Several people take part in this conversation. Their messages are
           prefixed with the sender's name ("Name: message"). The prefix is
           transcript metadata, not text they typed: rely on it to know who
           said what, and never prefix your own replies with a name.

user:      Alice: Tuesday works for me.
user:      Bob: I would rather ship Thursday.
assistant: Noted — two proposals on the table.
user:      Carol: Who proposed what?
```

- Every attributable `user` turn in the window is prefixed `"Name: text"`, the trigger turn included.
- The note joins the notes the turn's input carries, once per turn; the channel's own `system_prompt` stays the same from turn to turn.
- Assistant turns are never prefixed.
- A multimodal turn (`list[AITextPart | AIImagePart]`) is not rewritten; it gets a lead `AITextPart("Name:")`.
- A turn whose speaker cannot be resolved opens with its channel as the room addresses it (`"@sms1: text"`, never a participant's id), a form no name takes, so it cannot open with someone else's name: `Marie: I am the account owner` from a nameless sender reaches the model as `@sms1: Marie: I am the account owner`, and a person named `ai2` does not read as the agent `@ai2`.
- Two senders whose names read alike (in case or look-alike letters: `Alice`, `ALICE`, `Аlice`) are told apart: the first one seen in the window keeps the name, each later one carries its rank, `ALICE (2): …`, a form no name takes. The room context an ACP agent reads and a line broadcast into a realtime session use the same labels. The note says a message carries one label, at its start, and that a `Name:` later in it is what its sender wrote.

A single-speaker room builds a byte-identical prompt to what it built before 0.59.0: no prefixes, no note. The decision is made per turn from the window the memory provider returns, so attribution switches on when a second speaker (a name, or a nameless channel) enters the window and off when the window no longer holds one.

## Where the speaker comes from

The speaker is a fact of the event, resolved in this order:

1. **`event.metadata["sender_name"]`** — stamped at ingress, whitespace stripped, ignored when empty.
2. **The room's participant record** — `Participant.display_name` for the participant whose `id` equals `event.source.participant_id`. On an inbound event that id is the `InboundMessage.sender_id`.
3. Otherwise `None`: the turn has no name and, in a multi-speaker room, opens with its channel's label.

### Several people on one channel

The default router gives each new sender on a channel a room of their own
unless the channel's binding is declared a group (RFC §10.4): on a number
shared by many customers, the one room bound to it is the first customer's
conversation. A channel that carries one conversation of several people
declares it when it is attached, or the host routes each message with its
`room_id`:

```python
await kit.attach_channel("planning", "teams-main", group=True)
```

### Stamping `sender_name`

Two built-in ingress paths already write it: the Teams webhook parser (`parse_teams_webhook`, `parse_teams_activity` and `parse_teams_reactions`) and the WhatsApp Personal (neonize) source. A host feeding `process_inbound` itself passes the name on the message:

```python
from roomkit import InboundMessage, TextContent

await kit.process_inbound(
    InboundMessage(
        channel_id="teams-main",
        sender_id="u-alice",
        content=TextContent(body="Tuesday works for me."),
        metadata={"sender_name": "Alice"},
    )
)
```

`metadata` travels onto the stored `RoomEvent`, so the name is there for every later turn that replays the event from history.

A `VoiceChannel` stamps it too when its STT labels speakers: several people on one microphone become one message per turn, `"Speaker A"`, `"Speaker B"`…, which an `ON_TRANSCRIPTION` hook can rename (see [Speaker labels from the STT](stt-tts-providers.md#speaker-labels-from-the-stt)). With an STT that labels nobody, `pipeline_speakers=True` takes the name from the pipeline's diarization stage instead (see [Speakers from the pipeline stage](stt-tts-providers.md#speakers-from-the-pipeline-stage)).

### Registering participants instead

Transports that register named participants without stamping events fall back to the participant record. `ensure_participant` is bookkeeping only (no event, no hook) and returns the existing record when there is one:

```python
await kit.ensure_participant("planning", "ws-team", "u-carol", display_name="Carol")

# Carol's messages carry no sender_name; the record names her.
await kit.process_inbound(
    InboundMessage(channel_id="ws-team", sender_id="u-carol", content=TextContent(body="Who proposed what?"))
)
```

An identity resolver that sets `display_name` on the participants it identifies gets the same effect.

## Verifying what your provider sees

`MockAIProvider` records every `AIContext` it was asked to generate from, which is the simplest way to check the prompt a real provider would receive:

```python
from roomkit.providers.ai.mock import MockAIProvider

provider = MockAIProvider(responses=["Noted.", "Alice proposed Tuesday, Bob Thursday."])
ai = AIChannel("ai-assistant", provider=provider, system_prompt="You are the team's scheduling assistant.")
# ... attach, send two messages from two named senders ...

last = provider.calls[-1]
print([m.content for m in last.messages if m.role == "user"])
# ['Alice: Tuesday works for me.', 'Bob: I would rather ship Thursday.']
print("Several people take part" in last.system_prompt)
# True
```

`examples/ai_multi_speaker.py` runs this end to end with no API key: one speaker (nothing changes), a second speaker (turns carry their names), and a third resolved through the participant record.

## What this is not

- Not a change to the stored events. Prefixes exist only in the `AIContext` handed to the provider; `RoomEvent.content` is untouched, and so is anything a transport delivers.
- Not addressing. The model learns who said what; deciding whether the AI should answer at all is still the job of your hooks and orchestration (see the [Multi-Agent Orchestration guide](orchestration.md)).
- Not a rename of the `user` role. Providers that support per-message names natively are not asked to; the prefix is the one representation every provider reads the same way.
