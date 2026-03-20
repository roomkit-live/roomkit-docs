# Audio Pipeline

Audio processing pipeline for voice channels. See the [Audio Pipeline Stages guide](../guides/audio-pipeline-stages.md) for usage examples.

## Pipeline

::: roomkit.AudioPipeline

::: roomkit.AudioPipelineConfig

::: roomkit.AudioFrame

## VAD (Voice Activity Detection)

::: roomkit.VADProvider

::: roomkit.VADConfig

::: roomkit.VADEvent

::: roomkit.VADEventType

::: roomkit.MockVADProvider

## Denoiser

::: roomkit.DenoiserProvider

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserConfig

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserProvider

::: roomkit.MockDenoiserProvider

## Diarization

::: roomkit.DiarizationProvider

::: roomkit.DiarizationResult

::: roomkit.MockDiarizationProvider

## TTS Stream Filters

::: roomkit.TTSStreamFilter

::: roomkit.StripBrackets

::: roomkit.StripInternalTags

## Events & Callbacks

::: roomkit.SpeakerChangeEvent

::: roomkit.BargeInCallback

::: roomkit.AudioReceivedCallback
