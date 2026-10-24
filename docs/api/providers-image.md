# Image Providers

Image generation is its own provider surface (RFC §25), not a mode of the
conversational response. The model that holds a conversation is rarely one that
draws, so an agent conversing through Anthropic draws through Gemini exactly as
it transcribes through Deepgram.

::: roomkit.providers.image.ImageProvider

::: roomkit.providers.image.ImageResult

::: roomkit.providers.image.ImageOptions

::: roomkit.providers.image.ImageCapabilities

::: roomkit.providers.image.ImageModelInfo

::: roomkit.providers.image.ImageAttempt

::: roomkit.providers.image.ImageGenerationError

::: roomkit.providers.image.MockImageProvider

::: roomkit.providers.image.parse_size

::: roomkit.providers.image.to_data_uri

::: roomkit.providers.image.parse_data_uri

::: roomkit.providers.image.sniff_mime_type

## Vendor implementations

::: roomkit.providers.openai.OpenAIImageProvider

::: roomkit.providers.openai.OpenAIImageConfig

::: roomkit.providers.gemini.GeminiImageProvider

::: roomkit.providers.gemini.GeminiImageConfig

::: roomkit.providers.xai.XAIImageProvider

::: roomkit.providers.xai.XAIImageConfig

::: roomkit.providers.meta.MetaImageProvider

::: roomkit.providers.meta.MetaImageConfig

::: roomkit.providers.openrouter.OpenRouterImageProvider

::: roomkit.providers.openrouter.OpenRouterImageConfig

::: roomkit.providers.azure.AzureImageProvider

::: roomkit.providers.azure.AzureImageConfig

## Catalogs and pricing

Each provider ships an offline catalog through `available_models()`, disjoint
from the conversational catalog returned by `AIProvider.available_models()`: no
id draws *and* converses, so merging the two would only oblige every consumer of
the conversational list to filter out models it can never use. Entries carry
`capabilities=["image_gen", "edit"]` for a consumer that combines the lists on
purpose.

The usage counters an `ImageResult` reports — `input_tokens`,
`input_image_tokens`, `output_tokens`, `output_image_tokens` — are disjoint: a
token appears under exactly one of them, so summing them counts it once.

Pricing is separate from that report, and optional. The OpenAI and Gemini
lineups meter per token, with generated pixels on their own counter at a rate
an order of magnitude above text, so [`ModelPricing`](providers-ai.md) carries
`image_input_per_million` and `image_output_per_million` alongside the text
rates and `cost_for()` applies them. xAI and most of OpenRouter's lineup charge
a flat amount per image; their catalog entries carry no `pricing` rather than a
rate in the wrong unit, and OpenRouter reports the billed amount itself, which
its provider surfaces as `ImageResult.usage["cost"]`.

```python
entry = next(m for m in provider.available_models() if m.id == provider.model_name)
cost = entry.pricing.cost_for(result.usage)
```

See the [Image Generation guide](../guides/image-generation.md) for the whole
path, from a tool call to an image in a room.
