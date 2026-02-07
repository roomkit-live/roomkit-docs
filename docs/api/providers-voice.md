# Voice Providers

## Voice Backend

::: roomkit.voice.backends.base.VoiceBackend

::: roomkit.VoiceCapability

::: roomkit.VoiceSession

::: roomkit.VoiceSessionState

::: roomkit.AudioChunk

::: roomkit.TranscriptionResult

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

## RTP Backend

::: roomkit.voice.backends.rtp.RTPVoiceBackend

Install with: `pip install roomkit[rtp]`

## Mock Voice Backend

::: roomkit.voice.backends.mock.MockVoiceBackend

::: roomkit.voice.backends.mock.MockVoiceCall

## Voice Events

::: roomkit.BargeInEvent

::: roomkit.TTSCancelledEvent

::: roomkit.PartialTranscriptionEvent

::: roomkit.VADSilenceEvent

::: roomkit.VADAudioLevelEvent

## Callback Types

| Callback | Signature |
|----------|-----------|
| `SpeechStartCallback` | `(VoiceSession) -> Any` |
| `SpeechEndCallback` | `(VoiceSession, bytes) -> Any` |
| `PartialTranscriptionCallback` | `(VoiceSession, str, float, bool) -> Any` |
| `VADSilenceCallback` | `(VoiceSession, int) -> Any` |
| `VADAudioLevelCallback` | `(VoiceSession, float, bool) -> Any` |
| `BargeInCallback` | `(VoiceSession) -> Any` |
