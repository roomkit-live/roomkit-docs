# Meta Model API Providers

The [Meta Model API](https://dev.meta.ai/docs/overview) serves three models RoomKit
uses, on one key (`https://api.meta.ai/v1`, sent as a bearer token; Meta's docs
name it `MODEL_API_KEY`, RoomKit's examples read `META_API_KEY`):

| Model | RoomKit provider | What it does | Extra |
|-------|------------------|--------------|-------|
| `muse-spark-1.3` (and 1.2, 1.1) | `MetaAIProvider` | Chat, tools, images in, 1M-token window | `roomkit[meta]` |
| `muse-image-1.0` | `MetaImageProvider` | Image generation and editing | `roomkit[meta]` |
| `muse-voice-transcribe-1.0` | `MetaSTTProvider` | Streaming and batch speech-to-text, with speaker labels | `roomkit[meta-stt]` |

Each provider sits with the contract it implements: the chat and image providers in
`roomkit.providers.meta`, the STT beside the other recognisers in
`roomkit.voice.stt.meta`. Meta has no text-to-speech and no speech-to-speech
model. `muse-glimmer-30b` is open weights and not served by the API: reach it
through a self-hosted server with the vLLM, llama.cpp or Ollama providers.

The account needs a payment method before any billable call: without one every
request answers `402 billing_not_configured`, and the realtime STT socket closes
with 1008.

## Muse Spark (chat)

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.meta import MetaAIProvider, MetaConfig

provider = MetaAIProvider(MetaConfig(api_key="...", reasoning_effort="low"))

kit = RoomKit()
kit.register_channel(AIChannel("assistant", provider=provider, system_prompt="..."))
```

`MetaAIProvider` subclasses `OpenAIAIProvider` on Meta's Chat Completions:
message building, tool calls, streaming, usage (reasoning and cached tokens) and
errors are inherited. What is Meta's own:

- **Reasoning cannot be turned off.** `reasoning_effort` takes `minimal`, `low`,
  `medium`, `high` or `xhigh` and rides every request, tool turns included —
  it is the only lever over the cost and latency of a turn. `"none"` is refused
  by the service and sent as `"minimal"`. Measured on a one-line answer
  (2026-09-27): 3.0 s at `minimal`, 4.6 s at `low`; streaming at `low` showed
  its first text after ~7 s, the reasoning coming first. The reasoning is
  counted in `reasoning_tokens`, never returned.
- **`list_models()`** keeps the `muse-spark-*` ids: Meta's `/v1/models` also
  lists the image, speech and segmentation models.
- **Contributor ids** (`muse-spark-1.3-contributor`, `-1.2-contributor`) cost
  $0.10 / $0.20 per million tokens instead of $1.25 / $4.25, because Meta trains
  its models on their prompts and completions. They are never a default.

`examples/meta_ai.py` is a terminal chat on it (`--effort minimal` for speed).

## Muse Image

`MetaImageProvider` is covered in [Image Generation](image-generation.md): its
web search, image search and shell tools are sent **off** unless asked, `size`
sets the aspect ratio only, and the default format is WebP.

## Muse Voice Transcribe (speech-to-text)

`MetaSTTProvider` is covered in
[STT & TTS Providers](stt-tts-providers.md#meta-muse-voice-transcribe-cloud-api-streaming-batch):
the `ENDPOINTING` mode lets the model end each turn itself (about 550 ms of
silence), and `DIARIZATION` labels each turn's speaker, which a continuous
`VoiceChannel` carries into the room.
