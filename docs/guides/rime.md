# Rime: STT, TTS and Prism

Install `pip install 'roomkit[rime]'` and supply `RIME_API_KEY` from the environment.
The three providers implement RoomKit's existing voice contracts independently:

| Provider | Use | Audio and language |
|----------|-----|--------------------|
| `RimeSTTProvider` | One utterance, with revised partials | Mono PCM16 at 16 kHz; English or Spanish |
| `RimeTTSProvider` | Coda or Mist v3 synthesis, including streamed text input | Mono PCM16 at 8/16/24/48 kHz; language depends on model and voice |
| `RimeRealtimeProvider` | Prism speech-to-speech conversations | 16 kHz input, 24 kHz output; English |

## Transcription

```python
import os

from roomkit.voice.stt import RimeSTTConfig, RimeSTTProvider

stt = RimeSTTProvider(RimeSTTConfig(
    api_key=os.environ["RIME_API_KEY"],
    language="en",                 # en or es; no automatic detection
    mode="written",                # verbatim preserves spoken wording and fillers
    context_terms=["RoomKit"],
))

result = await stt.transcribe(audio)  # AudioFrame, AudioChunk or a WAV data URL
async for result in stt.transcribe_stream(audio_chunks, language="es"):
    # Each partial replaces the previous hypothesis; it is not a text delta.
    update_transcript(result.text, final=result.is_final)
```

Other PCM rates and stereo input are converted to 16 kHz mono. Remote audio
URLs must be fetched separately. Input ends when the iterator finishes or
yields an `is_final` chunk. Only the service's `done` event becomes a final
transcript; a lost connection raises instead of promoting the last partial.
The final language is the one the service reports.

Rime does not detect utterance boundaries. With `VoiceChannel`, configure a
pipeline VAD, or use `batch_mode=True` and call `flush_stt()` explicitly.
Do not use continuous STT mode. There is no diarization or confidence score.
Each utterance owns a connection, limited to 120 seconds including processing
and 16 MiB of PCM. `final_timeout_s` (default 10 seconds) bounds the wait after
the end marker. Context terms joined with `", "` must fit in 1024 UTF-8 bytes.

## Synthesis

```python
from roomkit.voice.tts import RimeTTSConfig, RimeTTSProvider

tts = RimeTTSProvider(RimeTTSConfig(
    api_key=os.environ["RIME_API_KEY"],
    model="coda",                  # or mistv3
    voice_id="astra",
    language="en",
    sample_rate=24000,
    time_scale_factor=1.0,          # lower is faster, higher is slower
))

audio = await tts.synthesize("Hello!")  # WAV data URL, duration and transcript
async for chunk in tts.synthesize_stream("Hello!"):
    await play_pcm(chunk)
async for chunk in tts.synthesize_stream_input(llm_text_stream):
    await play_pcm(chunk)
```

Text fragments retain their spaces and punctuation. Audio arrives while text
is still being supplied. Each synthesis has an independent socket and context;
closing its iterator cancels only that synthesis. `timeout_s` bounds the whole
synthesis, including time waiting for text.

Choose a voice matching the model and language. `available_voices()` supplies
a starter English voice; `await tts.list_voices(language="fr")` queries the
public catalog for the configured model. For example, Coda supports
`voice_id="aurelie", language="fr"`. Dialogue context and cloning are unsupported.
Use these providers in a `VoiceChannel` with your existing backend, VAD and AI
channel. `examples/voice_rime.py` synthesizes a sentence and transcribes it
with the same Rime key, without a microphone or separate LLM.

## Prism speech-to-speech

```python
from roomkit import RealtimeVoiceChannel
from roomkit.voice.realtime import RimeRealtimeConfig, RimeRealtimeProvider

provider = RimeRealtimeProvider(RimeRealtimeConfig(
    api_key=os.environ["RIME_API_KEY"],
    voice_id="astra",              # astra, aurora, brando, wawona
    eagerness="auto",
    end_of_turn_timeout_ms=5000,
))
channel = RealtimeVoiceChannel(
    "voice", provider=provider, transport=transport,
    input_sample_rate=16000, output_sample_rate=24000, server_vad=True,
    system_prompt="Keep your spoken replies brief.",
)
```

Prism owns turn detection. The adapter paces input in 20 ms frames and fills
capture gaps with silence. Keep microphone capture active during playback for
barge-in; use headphones or acoustic echo cancellation. Buffering is limited
to five seconds: supply live audio, not a recording uploaded faster than real
time. Run `examples/realtime_voice_local_rime.py` for a microphone conversation.

The provider maps transcripts, audio, speech/response boundaries, errors,
usage and function calls onto the realtime callbacks. Late user transcripts
remain independent of the assistant's reply. Tool results are submitted once;
continuation waits for every result's acknowledgement and the parent reply's
completion. Cancelling a response does not undo tools already issued.

Text injection preserves intent: user text becomes a typed user turn, system
text becomes an application instruction, and assistant text becomes an
instruction to speak the quoted line. Silent injection appends history.
`sent` means the service acknowledged the command; a missing acknowledgement
returns `unknown` to avoid inviting a replay of a possibly accepted turn.

`reconfigure()` updates instructions, tools, voice and `interrupt_response` on
the same connection. `provider_config` also accepts `eagerness`,
`end_of_turn_timeout_ms` and `truncation`; eagerness and end-of-turn timeout are
fixed at setup. Temperature and client endpointing are explicitly rejected.

A dropped connection is reported and closed without automatic reconnection
that would silently reset the conversation. Context preservation is not
advertised because Prism can evict old turns. RoomKit's `full_duplex` stays
false: Prism exposes response boundaries and native function calls.

## Verification and lifecycle

```bash
uv run pytest tests/voice/stt/test_rime.py tests/voice/tts/test_rime.py tests/voice/realtime/test_rime.py
```

These contract tests use local WebSocket servers. `tests/voice/test_rime_live.py`
is skipped unless `RIME_API_KEY` is set; enabling it makes billable service calls.
Close providers with `await close()` when using them directly. Channels manage
their lifetime in a RoomKit app.

Protocol references: [STT](https://docs.rime.ai/api-reference/stt/websockets),
[Coda](https://docs.rime.ai/api-reference/coda/websocket-v1),
[Mist v3](https://docs.rime.ai/api-reference/mistv3/websocket-v1),
[Prism](https://docs.rime.ai/api-reference/prism/websocket).
