# Image Generation

An agent that can describe a picture cannot draw one. `ImageProvider` (RFC §25) is the surface that lets it — and it is deliberately **separate from the AI provider**, the way speech-to-text and text-to-speech already are.

```python
from roomkit.providers.image import MockImageProvider

images = MockImageProvider()
results = await images.generate("un renard en origami", size="1024x1024")

results[0].data        # "data:image/png;base64,iVBORw0KGgo…"
results[0].mime_type   # "image/png"
results[0].decoded()   # b"\x89PNG\r\n…"
```

## Why it is not part of the conversation

Reaching image generation *through* the conversational response would tie the capability to the provider holding the conversation. It would then exist only for the subset of conversations already run by a model that draws, and acquiring it would mean migrating the conversation — a room at a time.

Decoupled, an agent conversing through Anthropic draws with a Gemini key, exactly as it transcribes with a Deepgram one. That is the arrangement roomkit already uses for every non-conversational capability, and it is what the RFC requires here (§25.1).

The second reason is narrower and just as decisive: `AIResponse.content` is a `str` read by every consumer of the framework. Turning it into a union of parts would impose a migration on all of them for a capability two providers offer.

## Providers

| Provider | Endpoint | Models | Extra |
|----------|----------|--------|-------|
| `OpenAIImageProvider` | `/v1/images` (`images.generate`, `images.edit`) | `gpt-image-2`, `gpt-image-1.5`, `gpt-image-1`, `gpt-image-1-mini`, `chatgpt-image-latest` | `roomkit[openai]` |
| `GeminiImageProvider` | Interactions API (`interactions.create`) | `gemini-3-pro-image`, `gemini-3.1-flash-image`, `gemini-3.1-flash-lite-image`, `gemini-2.5-flash-image` | `roomkit[gemini]` |
| `MockImageProvider` | — | `mock-image` | none — draws a real 1×1 PNG |

```python
from roomkit.providers.openai import OpenAIImageConfig, OpenAIImageProvider

images = OpenAIImageProvider(
    OpenAIImageConfig(api_key=..., model="gpt-image-2", quality="high")
)
```

```python
from roomkit.providers.gemini import GeminiImageConfig, GeminiImageProvider

images = GeminiImageProvider(
    GeminiImageConfig(api_key=..., model="gemini-3.1-flash-image")
)
```

Vendor-specific knobs live on the config, not on `generate()`: OpenAI's `quality` / `background` / `output_format`, Gemini's `image_size` / `output_mime_type`. The call itself stays the same shape everywhere.

## One size string, every vendor

`generate(size=...)` takes `"WIDTHxHEIGHT"` and the provider translates. OpenAI takes it verbatim from a fixed list. Gemini speaks aspect ratios and resolution tiers, so the provider reduces the fraction and picks the smallest tier that covers the request:

| `size` | OpenAI | Gemini |
|--------|--------|--------|
| `1024x1024` | `1024x1024` | `1:1`, `1K` |
| `1536x1024` | `1536x1024` | `3:2`, `2K` |
| `1920x1080` | rejected | `16:9`, `2K` |
| `3840x2160` | rejected | `16:9`, `4K` |

A size a provider cannot produce **raises** rather than becoming a different one silently — an image of the wrong geometry is a failure the caller can neither see nor correct.

## Editing

Editing is `generate()` with references, not a second method:

```python
[original] = await images.generate("un renard en origami")

[edited] = await images.generate(
    "make the paper blue",
    reference_images=[original.to_image_part()],
)
```

Under the hood the two vendors disagree — OpenAI routes to `images.edit` with a multipart upload, Gemini puts the images in the same call's content — and the provider absorbs it. A provider that cannot edit reports `supports_editing == False` and raises when handed references, rather than quietly generating from the prompt alone.

## The data-URI invariant

`ImageResult.data` is **always** a `data:<mime>;base64,<payload>` URI. Never bare base64, never a remote link. That removes the sniffing every consumer would otherwise have to do, and it makes the result immediately usable:

```python
from roomkit.models.event import MediaContent

await kit.send_event(
    room_id="atelier",
    channel_id="bot",
    content=MediaContent(url=result.data, mime_type=result.mime_type),
    addressed_to=[],   # the picture answers the turn; it does not start a new one
)
```

`MediaContent.url` accepts `data:` URIs, so the generated image enters the room with no conversion step, and `result.to_image_part()` hands it back for the next edit.

!!! warning "Address the injected image to nobody"
    Posting the image with `addressed_to=[]` (RFC §19.3) is what stops the loop: left unaddressed, the media message re-enters as a fresh prompt, the agent draws again, and the room fills up.

## Cost

Image models are billed **per token**, not per image — with the pixels on their own meter, at roughly an order of magnitude above the text rate. The counters an `ImageResult` reports are disjoint, so pricing is a single call:

```python
entry = next(m for m in type(images).available_models() if m.id == images.model_name)
cost = entry.pricing.cost_for(result.usage)   # USD
```

`result.usage` carries `input_tokens`, `input_image_tokens`, `output_tokens` and `output_image_tokens`; each token is counted exactly once, so summing them bills the call once. Where a vendor reports image tokens nested inside a total, the provider subtracts them before reporting.

The catalogs read the vendors' own price lists (verified 2026-08-07). Google's advertised per-image figures are that same token rate restated — a 1K image is 1120 output-image tokens, which at `$120`/M is the `$0.134` Nano Banana Pro advertises.

## Catalogs are disjoint

`ImageProvider.available_models()` is a separate list from `AIProvider.available_models()`. The two sets never intersect: `gpt-image-2` does not converse, `gpt-5.6-sol` does not draw. Merging them would save no maintenance — the entries are written once either way — while forcing every consumer of the conversational catalog, including anything populating a model picker for a chat agent, to filter out models it can never use.

Entries are still tagged `capabilities=["image_gen", "edit"]`, so a consumer that combines the lists on purpose can tell them apart.

## Testing without a key

`MockImageProvider` returns a real (tiny) PNG, so a consumer's tests exercise the whole path — decode, write, measure — offline:

```python
from roomkit.providers.image import MockImageProvider

images = MockImageProvider(supports_editing=False)   # or a list of your own bytes
[result] = await images.generate("anything")
assert result.decoded().startswith(b"\x89PNG")
assert images.calls == [("anything", None, 1, [])]
```

## End to end

`examples/image_generation.py` runs the whole path: an `AIChannel` calls a `generate_image` tool, the tool draws through the `ImageProvider`, and the picture lands in the room as `MediaContent` and on disk as a PNG. It runs with no key at all (mock), and with `OPENAI_API_KEY` or `GEMINI_API_KEY` it draws for real.

## What this is not

Image generation defines no hook triggers and is not a pipeline stage. It is called by a tool handler or an application, and its result enters a room as ordinary message content — which the inbound and broadcast pipelines already govern. Video generation and avatar synthesis are elsewhere; image *understanding* is already covered by multimodal message parts (`AIImagePart`).
