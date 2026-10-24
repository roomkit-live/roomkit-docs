# Audio Pipeline

Audio processing pipeline for voice channels. See the [Audio Pipeline Stages guide](../guides/audio-pipeline-stages.md) for usage examples.

## Pipeline

::: roomkit.voice.pipeline.AudioPipeline

::: roomkit.voice.pipeline.AudioPipelineConfig

::: roomkit.voice.AudioFrame

## VAD (Voice Activity Detection)

::: roomkit.voice.pipeline.vad.VADProvider

::: roomkit.voice.pipeline.vad.VADConfig

::: roomkit.voice.pipeline.vad.VADEvent

::: roomkit.voice.pipeline.vad.VADEventType

::: roomkit.voice.pipeline.vad.MockVADProvider

## Acoustic Echo Cancellation

::: roomkit.voice.pipeline.aec.AECProvider

::: roomkit.voice.pipeline.aec.WebRTCAECProvider

::: roomkit.voice.pipeline.aec.SpeexAECProvider

## Automatic Gain Control

::: roomkit.voice.pipeline.agc.AGCConfig

::: roomkit.voice.pipeline.agc.AGCProvider

::: roomkit.voice.pipeline.agc.SimpleAGCProvider

::: roomkit.voice.pipeline.agc.MockAGCProvider

## Denoiser

::: roomkit.voice.pipeline.denoiser.DenoiserProvider

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserConfig

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserProvider

::: roomkit.voice.pipeline.denoiser.RNNoiseDenoiserProvider

::: roomkit.voice.pipeline.denoiser.SherpaOnnxDenoiserProvider

::: roomkit.voice.pipeline.denoiser.WebRTCNoiseSuppressorProvider

::: roomkit.voice.pipeline.denoiser.MockDenoiserProvider

## Diarization

::: roomkit.voice.pipeline.diarization.DiarizationProvider

::: roomkit.voice.pipeline.diarization.DiarizationResult

::: roomkit.voice.pipeline.diarization.MockDiarizationProvider

## TTS Stream Filters

::: roomkit.voice.tts.TTSStreamFilter

::: roomkit.voice.tts.StripBrackets

::: roomkit.voice.tts.StripEmoji

::: roomkit.voice.tts.StripInternalTags

::: roomkit.voice.tts.StripTechnicalText

## Events & Callbacks

::: roomkit.voice.SpeakerChangeEvent

::: roomkit.voice.BargeInCallback

::: roomkit.voice.AudioReceivedCallback
