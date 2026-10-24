# Voice Providers

## Voice Backend

::: roomkit.voice.VoiceBackend

::: roomkit.voice.PlaybackErrors

::: roomkit.voice.VoiceCapability

::: roomkit.voice.VoiceSession

::: roomkit.voice.VoiceSessionState

::: roomkit.voice.AudioChunk

::: roomkit.voice.TranscriptionResult

::: roomkit.voice.SpeakerSegment

## STT (Speech-to-Text)

::: roomkit.voice.stt.STTProvider

::: roomkit.voice.stt.MockSTTProvider

::: roomkit.voice.STTLanguageLock

### Sherpa-ONNX STT

::: roomkit.voice.stt.SherpaOnnxSTTProvider

::: roomkit.voice.stt.SherpaOnnxSTTConfig

#### Usage

```python
from roomkit.voice.stt import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider

# Transducer model (streaming + batch)
stt = SherpaOnnxSTTProvider(SherpaOnnxSTTConfig(
    mode="transducer",
    tokens="path/to/tokens.txt",
    encoder="path/to/encoder.onnx",
    decoder="path/to/decoder.onnx",
    joiner="path/to/joiner.onnx",
))

# Whisper model (batch only)
stt = SherpaOnnxSTTProvider(SherpaOnnxSTTConfig(
    mode="whisper",
    tokens="path/to/tokens.txt",
    encoder="path/to/encoder.onnx",
    decoder="path/to/decoder.onnx",
    language="en",
))
```

Install with: `pip install roomkit[sherpa-onnx]`

### Gemini STT (Google, batch)

::: roomkit.voice.stt.GeminiSTTProvider

::: roomkit.voice.stt.GeminiSTTConfig

::: roomkit.voice.stt.GeminiTranscript

::: roomkit.voice.stt.GeminiTranscriptSegment

::: roomkit.voice.stt.GeminiTranscriptWord

#### Usage

```python
from roomkit.voice.stt import GeminiSTTConfig, GeminiSTTProvider

stt = GeminiSTTProvider(GeminiSTTConfig(api_key="your-gemini-api-key"))

transcript = await stt.transcribe_recording("meeting.wav")
for turn in transcript.segments:
    print(f"[{turn.start}-{turn.end}] {turn.speaker}: {turn.text}")
```

Batch only: the model takes a complete recording and answers in seconds, so this
fits meetings, voicemail and imported files rather than live turn-taking. Pass
`model="gemini-3.5-transcribe"` for Google's dedicated recogniser and its
word-level timing in `transcript.words` — see
the [STT & TTS Providers guide](../guides/stt-tts-providers.md#gemini-cloud-api-batch-only).

Install with: `pip install roomkit[gemini]`

### Meta Muse Voice Transcribe STT (streaming + batch)

::: roomkit.voice.stt.MetaSTTProvider

::: roomkit.voice.stt.MetaSTTConfig

::: roomkit.voice.stt.MetaSTTError

#### Usage

```python
from roomkit.voice.stt import MetaSTTConfig, MetaSTTProvider

# No pipeline VAD: the model finds where each turn ends
stt = MetaSTTProvider(MetaSTTConfig(api_key="your-meta-model-api-key"))

# Behind a pipeline VAD: one final per utterance
stt = MetaSTTProvider(MetaSTTConfig(api_key="...", mode="PUSH_TO_TALK"))

# Who spoke: every final names its speaker (read the provider directly)
stt = MetaSTTProvider(MetaSTTConfig(api_key="...", mode="DIARIZATION"))
async for result in stt.transcribe_stream(audio):
    if result.is_final:
        print(f"{result.speaker}: {result.text}")
```

See the [STT & TTS Providers guide](../guides/stt-tts-providers.md#meta-muse-voice-transcribe-cloud-api-streaming-batch)
for the mode choice and the limits.

Install with: `pip install roomkit[meta-stt]`

### Microsoft MAI-Transcribe STT (streaming)

::: roomkit.voice.stt.AzureMAISTTProvider

::: roomkit.voice.stt.AzureMAISTTConfig

::: roomkit.voice.stt.AzureMAISTTError

#### Usage

```python
from roomkit.voice.stt import AzureMAISTTConfig, AzureMAISTTProvider

# Behind a pipeline VAD: one final per utterance
stt = AzureMAISTTProvider(
    AzureMAISTTConfig(endpoint="https://<resource>.services.ai.azure.com", api_key="...")
)
```

See the [STT & TTS Providers guide](../guides/stt-tts-providers.md#microsoft-mai-transcribe-cloud-api-streaming)
for why it needs a pipeline VAD and the limits.

Install with: `pip install roomkit[azure-speech]`

### Mistral Voxtral STT (streaming + batch)

::: roomkit.voice.stt.VoxtralSTTProvider

::: roomkit.voice.stt.VoxtralSTTConfig

::: roomkit.voice.stt.VoxtralTurnDetection

::: roomkit.voice.stt.VoxtralSTTError

#### Usage

```python
from roomkit.voice.stt import VoxtralSTTConfig, VoxtralSTTProvider

# No pipeline VAD: the model finds where each turn ends
stt = VoxtralSTTProvider(VoxtralSTTConfig(api_key="your-mistral-api-key"))

# Behind a pipeline VAD: one final per utterance
stt = VoxtralSTTProvider(VoxtralSTTConfig(api_key="...", turn_detection=None))

# A recording, on the most accurate model
stt = VoxtralSTTProvider(VoxtralSTTConfig(api_key="...", model="voxtral-small-transcribe-3"))
result = await stt.transcribe(audio_content)

# Who spoke: every final names its speaker (continuous mode, no turn detection)
stt = VoxtralSTTProvider(VoxtralSTTConfig(api_key="...", diarize=True, turn_detection=None))
```

See the [Voxtral guide](../guides/voxtral.md) for the turn timing and the latency.

Install with: `pip install roomkit[mistral-voice]`

## TTS (Text-to-Speech)

::: roomkit.voice.tts.TTSProvider

::: roomkit.voice.tts.MockTTSProvider

### Voices and Dialogue

::: roomkit.voice.VoiceInfo

::: roomkit.voice.DialogueTurn

::: roomkit.voice.filter_voices

### Custom Voices

::: roomkit.voice.VoiceLibrary

::: roomkit.voice.CustomVoice

::: roomkit.voice.VoiceConsentError

### Conversation Context

::: roomkit.voice.tts.TTSContextLevel

::: roomkit.voice.tts.TTSContext

::: roomkit.voice.tts.ConversationTurn

::: roomkit.voice.tts.TTSContextConfig

### Vui Nano TTS

::: roomkit.voice.tts.VuiTTSProvider

::: roomkit.voice.tts.VuiTTSConfig

::: roomkit.voice.tts.VuiVoice

Install with: `pip install roomkit[vui]` (Python 3.12, CUDA GPU).

### Fluxions TTS (Hosted Vui)

::: roomkit.voice.tts.FluxionsTTSProvider

::: roomkit.voice.tts.FluxionsTTSConfig

Install with: `pip install roomkit[fluxions]`.

### Azure Speech TTS (MAI-Voice)

::: roomkit.voice.tts.AzureSpeechTTSProvider

::: roomkit.voice.tts.AzureSpeechTTSConfig

::: roomkit.voice.tts.AzureSpeechTTSError

Install with: `pip install roomkit[azure-speech]`.

### Fish Audio TTS (Drama 3)

::: roomkit.voice.tts.FishAudioTTSProvider

::: roomkit.voice.tts.FishAudioTTSConfig

::: roomkit.voice.tts.FishAudioTTSError

See the [STT & TTS Providers guide](../guides/stt-tts-providers.md#fish-audio-cloud-api-drama-3)
for expressive mode, the texts never sent and the measured limits.

Install with: `pip install roomkit[fish-audio]`.

### Pocket TTS

::: roomkit.voice.tts.PocketTTSProvider

::: roomkit.voice.tts.PocketTTSConfig

Install with: `pip install roomkit[pocket-tts]` (CPU or CUDA GPU).

### Phonon TTS

::: roomkit.voice.tts.PhononTTSProvider

::: roomkit.voice.tts.PhononTTSConfig

See the [Phonon guide](../guides/phonon.md) for models, voices and measured speed.

Install with: `pip install roomkit[phonon]` (CPU).

### Voxtral Speech TTS (Mistral)

::: roomkit.voice.tts.VoxtralTTSProvider

::: roomkit.voice.tts.VoxtralTTSConfig

::: roomkit.voice.tts.VoxtralTTSError

See the [Voxtral guide](../guides/voxtral.md) for the voices and the tags.

Install with: `pip install roomkit[mistral-voice]`

### Sherpa-ONNX TTS

::: roomkit.voice.tts.SherpaOnnxTTSProvider

::: roomkit.voice.tts.SherpaOnnxTTSConfig

#### Usage

```python
from roomkit.voice.tts import SherpaOnnxTTSConfig, SherpaOnnxTTSProvider

# VITS/Piper model with multi-speaker support
tts = SherpaOnnxTTSProvider(SherpaOnnxTTSConfig(
    model="path/to/vits-model.onnx",
    tokens="path/to/tokens.txt",
    data_dir="path/to/espeak-ng-data",  # for Piper models
    speaker_id=0,
    speed=1.0,
))
```

Install with: `pip install roomkit[sherpa-onnx]`

### Grok TTS (xAI)

::: roomkit.voice.tts.GrokTTSProvider

::: roomkit.voice.tts.GrokTTSConfig

#### Usage

```python
from roomkit.voice.tts import GrokTTSConfig, GrokTTSProvider

tts = GrokTTSProvider(GrokTTSConfig(
    api_key="your-xai-api-key",
    voice_id="eve",
    codec="pcm",
    sample_rate=24000,
))
```

Install with: `pip install httpx websockets`

### Gemini TTS (Google)

::: roomkit.voice.tts.GeminiTTSProvider

::: roomkit.voice.tts.GeminiTTSConfig

::: roomkit.voice.tts.GeminiVoiceLibrary

::: roomkit.voice.tts.GeminiVoiceLibraryConfig

#### Usage

```python
from roomkit.voice.tts import GeminiTTSConfig, GeminiTTSProvider

tts = GeminiTTSProvider(GeminiTTSConfig(
    api_key="your-gemini-api-key",
    model="gemini-3.8-flash-tts",
    voice="Kore",
    style_prompt="calm and clear",                 # delivery guidance
))
```

`list_voices()` reads Google's catalog of about 2,000 voices, `synthesize_dialogue()`
voices two speakers in one clip, and `GeminiVoiceLibrary` designs and replicates
custom voices.

Time to first audio is measured in seconds, so this fits prompts and generated
audio messages rather than live turn-taking — see the
[STT & TTS Providers guide](../guides/stt-tts-providers.md#gemini-tts-cloud-api).

Install with: `pip install roomkit[gemini]`

## RTP Backend

::: roomkit.voice.backends.RTPVoiceBackend

Install with: `pip install roomkit[rtp]`

## Mock Voice Backend

::: roomkit.voice.backends.MockVoiceBackend

::: roomkit.voice.backends.MockVoiceCall

## Voice Events

::: roomkit.voice.BargeInEvent

::: roomkit.voice.TTSCancelledEvent

::: roomkit.voice.PartialTranscriptionEvent

::: roomkit.voice.VADSilenceEvent

::: roomkit.voice.VADAudioLevelEvent

## Callback Types

| Callback | Signature |
|----------|-----------|
| `SpeechStartCallback` | `(VoiceSession) -> Any` |
| `SpeechEndCallback` | `(VoiceSession, bytes) -> Any` |
| `PartialTranscriptionCallback` | `(VoiceSession, str, float, bool) -> Any` |
| `VADSilenceCallback` | `(VoiceSession, int) -> Any` |
| `VADAudioLevelCallback` | `(VoiceSession, float, bool) -> Any` |
| `BargeInCallback` | `(VoiceSession) -> Any` |
