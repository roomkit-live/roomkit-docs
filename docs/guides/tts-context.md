# TTS Conversation Context

A TTS that synthesizes each sentence in isolation cannot carry prosody across a
conversation. Some can, when they are told what came before: their own previous
generations, the text of the dialogue, or its audio, including how the user
spoke. The `VoiceChannel` keeps that dialogue per voice session and hands it to
a provider that declares it can use it (RFC §12.2.2).

## Declaring what a provider consumes

A provider says what it can use through `context_level`:

| Level | The provider receives | Typical use |
|---|---|---|
| `TTSContextLevel.NONE` | Nothing (default) | Most TTS engines |
| `TTSContextLevel.SELF` | Its own previous turns only, and how much of each was played | Continuity handles returned by a cloud API |
| `TTSContextLevel.TEXT` | The text of every turn, both roles | Previous / next text conditioning |
| `TTSContextLevel.AUDIO` | The text and the audio of every turn | Dialogue-conditioned models |

A provider left at `NONE` is called exactly as before: the channel never passes
it a `context`, so a provider written against the older signature keeps
working.

```python
from roomkit import TTSContext, TTSContextLevel
from roomkit.voice.tts.base import TTSProvider


class MyTTS(TTSProvider):
    @property
    def context_level(self) -> TTSContextLevel:
        return TTSContextLevel.TEXT

    async def synthesize_stream(self, text, *, voice=None, context: TTSContext | None = None):
        previous = [t.text for t in context.turns] if context else []
        ...  # condition the synthesis on `previous`

    def release_context(self, context_id: str) -> None:
        ...  # drop anything kept for this voice session
```

## What the context holds

`TTSContext` is a snapshot, oldest turn first:

- `context_id`: the voice session id. One context per session, never shared
  with another session or channel.
- `turns`: `ConversationTurn` records, each with `turn_id`, `role`
  (`"user"` or `"assistant"`), `participant_id`, `text`, and for assistant
  turns `played_ms` and `interrupted`. `audio` is set only at the `AUDIO`
  level with audio enabled.
- `next_turn_id`: the id the assistant turn of this call will get once it is
  played. A provider that needs its own handle for a turn (a request id) keeps
  it under that id and finds it in a later context.

User turns are recorded after `ON_TRANSCRIPTION`, with the text as the hooks
left it. Assistant turns are recorded when playback ends, with the text sent to
the TTS. On a barge-in the assistant turn is cut to what was played: its audio
stops at `played_ms`, the same value the timeline records for the interrupted
utterance, with or without `flush_partial_tts`. The turn is recorded at the
interruption, so the call that replaces it already sees it. A call cancelled
before any audio played leaves no turn. A speech
segment classified as a backchannel is not a turn.

In continuous STT mode there is no captured utterance, so user turns carry
text only.

## Configuring it

```python
from roomkit import TTSContextConfig, VoiceChannel

voice = VoiceChannel(
    "voice",
    stt=stt,
    tts=tts,
    backend=backend,
    tts_context=TTSContextConfig(
        include_audio=True,       # off by default
        max_turns=20,             # oldest turns dropped past this
        max_audio_seconds=120.0,  # oldest audio dropped past this, its text stays
    ),
)
```

`TTSContextConfig(enabled=False)` turns it off for a provider that would
otherwise receive it.

## Privacy

Context audio is a copy of what the user said (RFC §17.6):

- it is kept in memory only, never in the store, the timeline, a recording or
  a log;
- it stays within `max_turns` and `max_audio_seconds`, and is dropped when the
  session is unbound or the channel closed, together with a call to the
  provider's `release_context`; a turn that finishes after the session was
  unbound is not kept;
- it is off by default (`include_audio=False`);
- a user turn whose transcript a hook changed (a redaction) keeps no audio,
  and neither does the speech segment in which DTMF was detected while DTMF
  redaction is enabled.

A provider at the `AUDIO` level that runs as an external service receives the
user's voice on every call, not only the current sentence. Say so wherever you
document which services receive audio.

## Observability

Each call records `pipeline.tts_context_turns` and `pipeline.tts_context_audio_s`
through the telemetry provider.

## Example

`examples/voice_tts_context.py` runs without credentials: a toy provider logs
the dialogue it receives on each call.
