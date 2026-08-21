# Image Providers

Image generation is its own provider surface (RFC §25), not a mode of the
conversational response. The model that holds a conversation is rarely one that
draws, so an agent conversing through Anthropic draws through Gemini exactly as
it transcribes through Deepgram.

::: roomkit.providers.image.base.ImageProvider

::: roomkit.providers.image.base.ImageResult

::: roomkit.providers.image.mock.MockImageProvider

::: roomkit.providers.image.base.parse_size

::: roomkit.providers.image.base.to_data_uri

::: roomkit.providers.image.base.sniff_mime_type

## Vendor implementations

::: roomkit.providers.openai.image.OpenAIImageProvider

::: roomkit.providers.openai.config.OpenAIImageConfig

::: roomkit.providers.gemini.image.GeminiImageProvider

::: roomkit.providers.gemini.config.GeminiImageConfig

::: roomkit.providers.xai.image.XAIImageProvider

::: roomkit.providers.xai.config.XAIImageConfig

::: roomkit.providers.openrouter.image.OpenRouterImageProvider

::: roomkit.providers.openrouter.config.OpenRouterImageConfig

::: roomkit.providers.azure.image.AzureImageProvider

::: roomkit.providers.azure.config.AzureImageConfig

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
