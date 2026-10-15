# Images in AI Turns

An image a person sends reaches an AI channel's model as an image, with the
words of the message that brought it, on later turns too. This guide covers
what decides which images go, how an application that keeps its media in
private storage hands them over without publishing them, and what the model
reads when an image cannot go (RFC §6.4).

## Three kinds of URL

A `MediaContent` URL is one of:

| URL | Example | What the AI channel does |
|-----|---------|--------------------------|
| A link | `https://cdn.example.com/lamp.jpg` | Hands it to the provider, which fetches it |
| A `data:` URI | `data:image/png;base64,iVBOR…` | Sends the bytes it carries |
| A reference | `media://photo-42` | Asks the application's `media_resolver` for the bytes |

A reference is `<scheme>://…` in a scheme your application chooses: RoomKit
never interprets it. A `file:`, `ftp:`, `javascript:` or `vbscript:` URL, a URL
without a scheme, and a reference a path reader could follow (a one-letter
scheme read as a Windows drive, a `..` segment, a backslash) are refused when
the `MediaContent` is built, because some transports read a URL they are
handed as a local path.

## Images by reference

An application that keeps its users' media in private storage stores a
reference in the event and gives the framework a resolver. Every AI channel
registered on the kit asks it for the bytes as it builds a turn; the store and
every broadcast keep the reference alone, never the bytes.

```python
from roomkit import InboundMessage, RoomKit
from roomkit.models.event import MediaContent


async def resolve(media: MediaContent, room_id: str) -> bytes | None:
    """The bytes of a media this room may show, or None."""
    owner, data = await storage.read(media.url)
    return data if owner == room_id else None


kit = RoomKit(media_resolver=resolve)

await kit.process_inbound(
    InboundMessage(
        channel_id="web",
        sender_id="alice",
        content=MediaContent(url="media://photo-42", mime_type="image/jpeg", filename="lamp.jpg"),
    )
)
```

**Authorise every reference.** Any participant can write any reference in a
message, so the resolver answers only for a media the room it is asked for may
show, and `None` for any other.

Nothing is cached between turns: the resolver is asked again on each turn the
image goes, at most four times per turn with the default bound. Cache on your
side if reading the storage is expensive.

A transport channel that takes media (MMS, WhatsApp) is handed the reference
as it is, which its vendor cannot fetch. Where a transport needs the media,
resolve the reference yourself, for example in a `BEFORE_BROADCAST` hook that
swaps it for a link the vendor can reach.

## Which images go

Per turn, the newest four images authors sent go to a model that sees, history
and input together, inline, linked or referenced alike. Each goes after its
caption, in the message of the person who sent it, before the images the
channel's tools returned (which ride the turn's input on their own bound).

```python
agent = AIChannel("assistant", provider=provider, max_author_images=2)    # the two newest
agent = AIChannel("assistant", provider=provider, max_author_images=None) # every image in the window
```

Only an image every vision provider takes goes as an image: PNG, JPEG, GIF or
WebP (`image/jpg`, capitals and parameters are read as the same types, and the
image is sent under the type it was read as: `image/jpg` goes as `image/jpeg`),
with data, up to 5 MB. Shrink larger photos before you store them, or in the
resolver. A link's or a `data:` URI's scheme is stored in lower case
(`HTTP://…` as `http://…`), the spelling every transport tests.

## What takes an image's place

An image that does not go is replaced by a short mention, never a failed turn:

| Case | The model reads |
|------|-----------------|
| A document (`application/pdf`…), or any media for a model without vision | `[Media: contract.pdf]` |
| An image past the bound | `[Earlier image not shown: “lamp.jpg”]` |
| A reference with no resolver, a resolver that fails or answers `None`, bytes too large | `[Image unavailable: “lamp.jpg”]` |
| An image a provider refused on an earlier turn | `[Image refused by the model's provider: “lamp.jpg”]` |

The name is the caption, else the filename, else the media type: a mention
never carries a `data:` URI or a reference.

When the turn's first request fails, before anything of it streamed, on a
provider error that is not retried and is no authentication, payment (a 402),
permission, missing model or rate limit (the provider could not process an image), the
author images it carried read as refused on the turns after it, so one bad
image does not fail every later turn while it stays in the history window. A
failure on a later round, or once the answer streams, leaves them: the
provider took them. The images the channel's tools returned are dropped the
same way.

## A model that does not see

A model without vision reads each media as its label (`[Media: lamp.jpg]`),
never its bytes, in the history and the turn's input alike: an AI channel takes
media whatever its model sees, so the router hands it the event untranscoded. A captionless media
alone in its event goes to `describe_empty_event` first, so a host keeps the
word on what such an upload says:

```python
def describe_upload(event):
    return "The member sent a photo without a caption."

agent = AIChannel("assistant", provider=text_only_provider, describe_empty_event=describe_upload)
```

## Example

`examples/ai_image_reference.py` keeps a colour swatch in a private store,
sends it by reference with a question, then asks a follow-up that needs the
image again. Offline it shows what a mock model received on each turn; with
`anthropic`, Claude names the colour, then the shape from the history.
