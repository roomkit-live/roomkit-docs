# Voice Providers

## Voice Backend

::: roomkit.voice.backends.base.VoiceBackend

::: roomkit.voice.base.VoiceCapability

::: roomkit.voice.base.VoiceSession

::: roomkit.voice.base.VoiceSessionState

::: roomkit.voice.base.AudioChunk

::: roomkit.voice.base.TranscriptionResult

## STT (Speech-to-Text)

::: roomkit.voice.stt.base.STTProvider

::: roomkit.voice.stt.mock.MockSTTProvider

### Sherpa-ONNX STT

::: roomkit.voice.stt.sherpa_onnx.SherpaOnnxSTTProvider

::: roomkit.voice.stt.sherpa_onnx.SherpaOnnxSTTConfig

#### Usage

```python
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTProvider, SherpaOnnxSTTConfig

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

## TTS (Text-to-Speech)

::: roomkit.voice.tts.base.TTSProvider

::: roomkit.voice.tts.mock.MockTTSProvider

### Sherpa-ONNX TTS

::: roomkit.voice.tts.sherpa_onnx.SherpaOnnxTTSProvider

::: roomkit.voice.tts.sherpa_onnx.SherpaOnnxTTSConfig

#### Usage

```python
from roomkit.voice.tts.sherpa_onnx import SherpaOnnxTTSProvider, SherpaOnnxTTSConfig

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

::: roomkit.voice.tts.grok.GrokTTSProvider

::: roomkit.voice.tts.grok.GrokTTSConfig

#### Usage

```python
from roomkit.voice.tts.grok import GrokTTSProvider, GrokTTSConfig

tts = GrokTTSProvider(GrokTTSConfig(
    api_key="your-xai-api-key",
    voice_id="eve",
    codec="pcm",
    sample_rate=24000,
))
```

Install with: `pip install httpx websockets`

### Gemini TTS (Google)

::: roomkit.voice.tts.gemini.GeminiTTSProvider

::: roomkit.voice.tts.gemini.GeminiTTSConfig

#### Usage

```python
from roomkit.voice.tts.gemini import GeminiTTSConfig, GeminiTTSProvider

tts = GeminiTTSProvider(GeminiTTSConfig(
    api_key="your-gemini-api-key",
    model="gemini-3.1-flash-tts-preview",
    voice="Kore",
    style_prompt="Read this calmly and clearly",   # delivery guidance
))
```

Time to first audio is measured in seconds, so this fits prompts and generated
audio messages rather than live turn-taking — see the
[STT & TTS Providers guide](../guides/stt-tts-providers.md#gemini-tts-cloud-api).

Install with: `pip install roomkit[gemini]`

## RTP Backend

::: roomkit.voice.backends.rtp.RTPVoiceBackend

Install with: `pip install roomkit[rtp]`

## Mock Voice Backend

::: roomkit.voice.backends.mock.MockVoiceBackend

::: roomkit.voice.backends.mock.MockVoiceCall

## Voice Events

::: roomkit.voice.events.BargeInEvent

::: roomkit.voice.events.TTSCancelledEvent

::: roomkit.voice.events.PartialTranscriptionEvent

::: roomkit.voice.events.VADSilenceEvent

::: roomkit.voice.events.VADAudioLevelEvent

## Callback Types

| Callback | Signature |
|----------|-----------|
| `SpeechStartCallback` | `(VoiceSession) -> Any` |
| `SpeechEndCallback` | `(VoiceSession, bytes) -> Any` |
| `PartialTranscriptionCallback` | `(VoiceSession, str, float, bool) -> Any` |
| `VADSilenceCallback` | `(VoiceSession, int) -> Any` |
| `VADAudioLevelCallback` | `(VoiceSession, float, bool) -> Any` |
| `BargeInCallback` | `(VoiceSession) -> Any` |
