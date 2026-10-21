# Mistral Voxtral 3 Speech

Mistral's Voxtral 3 family (October 2026, public preview) transcribes and speaks.
With `MistralAIProvider` for the answers, a RoomKit voice agent runs on Mistral end
to end, on one API key (`MISTRAL_API_KEY`).

| Model | Id | RoomKit provider | What it does |
|-------|----|------------------|--------------|
| Voxtral Mini Transcribe 3 | `voxtral-mini-transcribe-3` | `VoxtralSTTProvider.transcribe()` | File transcription (default) |
| Voxtral Small Transcribe 3 | `voxtral-small-transcribe-3` | `VoxtralSTTProvider.transcribe()` | File transcription, most accurate |
| Voxtral Mini Transcribe Realtime 3 | `voxtral-mini-transcribe-realtime-3` | `VoxtralSTTProvider.transcribe_stream()` | Live transcription with turn detection |
| Voxtral Speech 3 | `voxtral-mini-tts-3` | `VoxtralTTSProvider` | Text-to-speech, streamed |

Install with `pip install roomkit[mistral-voice]`. The providers speak Mistral's wire
protocol over `websockets` and `httpx`, with no Mistral SDK: the SDK's realtime
client (3.0.0) cannot send turn detection.

Voxtral 3 is served by Mistral's API only; its weights are not published. Mistral's
earlier open models (Voxtral Mini 4B Realtime 2602, Apache 2.0; Voxtral 4B TTS 2603,
CC BY-NC 4.0) run on vLLM, and are not what these providers call.

## Speech-to-text

```python
from roomkit.voice.stt.voxtral import VoxtralSTTConfig, VoxtralSTTProvider

stt = VoxtralSTTProvider(VoxtralSTTConfig(
    api_key="...",
    language="fr",                      # None: the model detects it; "fr-CA" is sent as "fr"
    context_bias=["Northwind", "Alder Creek Design"],  # names to spell right
))
```

### Files

`transcribe()` sends one clip to the file endpoint: PCM (an `AudioChunk` or an
`AudioFrame`) as a WAV, a `data:` URI as the file it carries (MP3, WAV, FLAC…), and
an http(s) URL as `file_url`, which Mistral fetches (RoomKit never does). Set
`model="voxtral-small-transcribe-3"` for the most accurate model. Mistral reports no
confidence and no language (`null` in every answer measured).

### Live: two ways a stream ends

How a stream ends depends on the `VoiceChannel`, which the provider cannot see, so it
is configuration:

| `turn_detection` | Channel | Finals |
|------------------|---------|--------|
| `VoxtralTurnDetection()` (default) | No pipeline VAD (continuous mode) | One per turn the model ends; each turn's start fires the speech-start hooks, and its words, as they arrive, can cut the agent off |
| `None` | Behind a pipeline VAD | One per utterance, once the stream ends |

The wrong choice loses words: behind a VAD with turn detection on, the channel keeps
only the last turn of an utterance; without a VAD and with it off, no final comes
until the audio stops.

```python
from roomkit import VoiceChannel
from roomkit.voice.pipeline import AudioPipelineConfig

# The model finds the turns: no VAD in the pipeline
voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend,
                     pipeline=AudioPipelineConfig())
```

Partials carry the text so far: Voxtral's text is final as it arrives and only ever
grows. The documented input rates (8, 16, 22.05, 24, 44.1, 48 kHz) go through as
they are; any other is resampled to 16 kHz.

### Turn detection and latency

The model scores how likely the person is still speaking, and its thresholds turn the
score into turn events. `VoxtralTurnDetection()` carries the service's defaults;
`VoxtralTurnDetection.aggressive()` is Mistral's faster preset. `streaming_delay_ms`
is how long the model listens past a word before writing it: any value from 80 to
1200, or 2400 (480 by default).

The turn events come at once, while the text runs a streaming delay behind the
audio: a turn's last words follow its `turn.end` by up to that delay, and even the
next `turn.start` (measured: "What is the capital of Italy?" had its last two words
arrive after the next turn had started). So a turn's final waits the delay plus
100 ms (`final_wait_ms`, 0.58 s by default), and the text arriving until then is
that turn's; a new turn's speech start is reported at once. When the next turn
starts during that wait, the final comes a little sooner, 0.95 of the delay after
the end: the ending turn's last words came up to 0.9 of the delay after it
(0.43 s at 480 ms), and the new turn's first word comes no sooner than a delay
after it — with the plain wait, that word had been joined to the last turn. Words
later than the wait are dropped, with a warning naming `final_wait_ms` when they
arrive.

With turn detection the provider reports `keeps_stream_across_turns`, so the
`VoiceChannel` keeps one realtime session for the whole conversation instead of
reconnecting after each turn. On a session per turn, a turn started within the
wait was sent half to the session being closed and reached the room in pieces
("What is the capital of Italy?", "Answer", "in one word.").

Measured on 2026-10-21, a spoken question fed in real time through a `VoiceChannel`:

| Settings | Final after the end of speech |
|----------|-------------------------------|
| Defaults (480 ms delay) | 1.7 s |
| `aggressive()`, `streaming_delay_ms=240` | 0.7 s |

The faster settings end a turn on a shorter pause: "What is the capital of Italy?
Answer in one word." came out as two turns, and the agent answered both.

## Speaker labels

`diarize=True` labels who spoke, in a file and live: the provider reports
`supports_diarization`, and every final names its speaker in `segments`
(RFC §12.2.3) — Mistral's `speaker_1` arrives as `"1"`, so a `VoiceChannel` names
the voice "Speaker 1".

```python
stt = VoxtralSTTProvider(VoxtralSTTConfig(
    api_key="...",
    diarize=True,
    turn_detection=None,        # live, Voxtral labels speakers or detects turns, not both
    speaker_references={"Julie": julie_wav, "Sylvain": sylvain_wav},  # optional, live
))
```

- **File:** one segment per run of a speaker, with its start and end (the
  endpoint returns a sentence or so per segment; consecutive ones of one speaker
  are joined).
- **Live:** the session sends a few words at a time with their speaker and no
  turn events, so a turn ends when another voice starts, or after a pause
  without new words (`diarized_pause_ms`, 1.5 s by default). One person's words
  came up to 2 s apart in hesitant speech and a new speaker's 1.5 s after the
  last (measured), so a long hesitation can end someone's turn in two.
- **Channel:** a `VoiceChannel` carries the labels in continuous mode (no pipeline
  VAD), keeping one session across turns; behind a VAD it refuses the provider.
- **Speaker references** name known voices live: a 3 to 15 s recording of each
  person (WAV, MP3, FLAC or Opus, mostly speech), at most 8, under the name their
  words then carry. A voice the references do not match is numbered after them
  ("3"). Matching is the model's, and partial: in one measured session a voice
  came back under its name for its first words, then as "3". The file endpoint
  took references and kept its own labels, so the provider sends them live only.
  The channel names a label "Speaker Julie"; an `ON_TRANSCRIPTION` hook that sets
  `sender_name` to the label makes it "Julie" (the diarization example does).

[`examples/voice_voxtral_diarization.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_voxtral_diarization.py)
labels the voices around one microphone (`REFERENCES="Julie=julie.wav"`).

## Text-to-speech

```python
from roomkit.voice.tts.voxtral import VoxtralTTSConfig, VoxtralTTSProvider

tts = VoxtralTTSProvider(VoxtralTTSConfig(
    api_key="...",
    voice="voice_3137",      # Manon, French; default voice_478 (Sheila, US English)
    expressiveness=2.0,      # from 1.0; None keeps the service's default
))
```

Each text is rendered whole and streamed back as it renders (first audio about
0.4–0.5 s, measured). The audio is 24 kHz mono: the service sends 32-bit float
samples for `pcm` (not documented), which the provider converts to the 16-bit PCM
the voice path carries. `synthesize()` returns a 16-bit WAV. Mistral's realtime
speech model, which takes text while it is being written, was not served at launch,
so the channel splits a streamed reply into sentences for this provider.

### Voices

`VoxtralTTSProvider.available_voices()` lists the 36 Voxtral 3 presets offline, by
slug, with language and gender: English (US and UK), French (`voice_592` Marie,
`voice_3137` Manon, `voice_589` Lea, `voice_3141` Camille), and Arabic, Catalan,
German, Spanish, Hindi, Hungarian, Italian, Japanese, Korean, Dutch, Norwegian,
Polish, Portuguese, Romanian, Russian, Turkish, Ukrainian, Vietnamese and Chinese.
`await tts.list_voices()` asks the service: the same presets, Voxtral 2's
(`gb_jane_neutral`, `fr_marie_neutral`…, still accepted), and the voices saved in the
workspace, whose id is the UUID `voice` takes.

### Emotion and pace tags

Voxtral 3 voices follow inline tags in the text: `[emotion=happy]`, `[pace=slow]`,
`[pace=fast]`. A `StripBrackets` filter in front of the provider drops every
`[...]` unless told to keep it, so leave it out (or keep the tags) when the agent
writes them.

## Errors

What the service refuses or fails raises a `ProviderError` subclass
(`VoxtralSTTError`, `VoxtralTTSError`) carrying Mistral's `code` and `type`, and
`retryable` for a rate limit, a server fault, a realtime connection that drops, or a
speech render that ends before its end. An HTTP transport failure (connection
refused, timeout) comes through as httpx's own exception, as with RoomKit's other
HTTP providers. What the service answers:

| Case | Answer |
|------|--------|
| Wrong API key | 401 (the realtime socket refuses the upgrade) |
| Realtime model the key cannot use | 403 with no body |
| Streaming delay outside 80–1200, 2400 | `error` event, code 3051 (refused before connecting) |
| Unknown voice | 404, code `1902`, type `invalid_voice` |
| `expressiveness` below 1.0 | 422 (refused in the config) |

## Example

[`examples/voice_voxtral.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_voxtral.py)
runs the agent on the local microphone: Voxtral hears you, Mistral Small answers,
Voxtral Speech speaks. `TURNS=aggressive STREAMING_DELAY_MS=240` makes it answer
sooner; `VOICE=voice_3137 VOICE_LANGUAGE=fr` makes it French.

```bash
MISTRAL_API_KEY=... uv run --extra mistral-voice --extra mistral \
    --extra local-audio --extra webrtc-aec python examples/voice_voxtral.py
```

## Not covered yet

- The eager end of a turn (`turn.eager_end`, `turn.resume`), which would let the
  agent prepare its answer early: RoomKit's STT contract has no such signal yet.
- Voice cloning and voice management, and `phoneme_hints`.
- Mistral's Voice Agents SDK, a whole hosted agent stack on LiveKit: the layer
  RoomKit itself provides.
