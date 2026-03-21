# Audio Pipeline

Audio processing pipeline for voice channels. See the [Audio Pipeline Stages guide](../guides/audio-pipeline-stages.md) for usage examples.

## Pipeline

::: roomkit.voice.pipeline.engine.AudioPipeline

::: roomkit.voice.pipeline.config.AudioPipelineConfig

::: roomkit.voice.audio_frame.AudioFrame

## VAD (Voice Activity Detection)

::: roomkit.voice.pipeline.vad.base.VADProvider

::: roomkit.voice.pipeline.vad.base.VADConfig

::: roomkit.voice.pipeline.vad.base.VADEvent

::: roomkit.voice.pipeline.vad.base.VADEventType

::: roomkit.voice.pipeline.vad.mock.MockVADProvider

## Denoiser

::: roomkit.voice.pipeline.denoiser.base.DenoiserProvider

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserConfig

::: roomkit.voice.pipeline.denoiser.AICousticsDenoiserProvider

::: roomkit.voice.pipeline.denoiser.mock.MockDenoiserProvider

## Diarization

::: roomkit.voice.pipeline.diarization.base.DiarizationProvider

::: roomkit.voice.pipeline.diarization.base.DiarizationResult

::: roomkit.voice.pipeline.diarization.mock.MockDiarizationProvider

## TTS Stream Filters

::: roomkit.voice.tts.filters.TTSStreamFilter

::: roomkit.voice.tts.filters.StripBrackets

::: roomkit.voice.tts.filters.StripInternalTags

## Events & Callbacks

::: roomkit.voice.events.SpeakerChangeEvent

::: roomkit.voice.base.BargeInCallback

::: roomkit.voice.backends.base.AudioReceivedCallback
