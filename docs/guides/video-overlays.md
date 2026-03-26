# Video Overlays

Render dynamic content — text, images, tables — onto live video frames. The overlay system plugs into the existing video pipeline as a `VideoFilterProvider`.

## Quick start

```python
from roomkit.video.pipeline.overlay import OverlayFilter, Overlay, TextOverlayRenderer

overlay = OverlayFilter(renderers=[TextOverlayRenderer()])
overlay.add_overlay(Overlay(id="title", content="Live Meeting"))

# Add to your video pipeline config
config = VideoPipelineConfig(filters=[overlay])
```

## Live subtitles

The killer use case: user speaks French → STT transcribes → AI translates → English subtitles on video.

```python
from roomkit.video.pipeline.overlay import subtitle_overlay

# One-liner: creates overlay + wires ON_TRANSCRIPTION hook
overlay = subtitle_overlay(kit, font_scale=0.8, color=(255, 255, 0))
config = VideoPipelineConfig(filters=[overlay])
```

### With translation

```python
async def translate(text: str) -> str:
    response = await ai.generate(f"Translate to English: {text}")
    return response.text

overlay = subtitle_overlay(kit, translate_fn=translate, max_lines=3)
```

### SubtitleManager (full control)

```python
from roomkit.video.pipeline.overlay import SubtitleManager, OverlayPosition

mgr = SubtitleManager(
    kit,
    translate_fn=translate,
    position=OverlayPosition.BOTTOM_CENTER,
    max_lines=2,
    style={"font_scale": 0.9, "bg_color": (0, 0, 0)},
)

# Add the filter to your pipeline
config = VideoPipelineConfig(filters=[mgr.overlay_filter])

# Manual control
mgr.set_text("Custom subtitle")
mgr.clear()
```

## Multiple overlays

Stack overlays with z-ordering:

```python
overlay = OverlayFilter(renderers=[TextOverlayRenderer()])

# Title at the top
overlay.add_overlay(Overlay(
    id="title", content="Live Meeting",
    position=OverlayPosition.TOP_CENTER,
    z_order=10,
))

# Subtitles at the bottom (higher z = on top)
overlay.add_overlay(Overlay(
    id="subtitle", content="",
    position=OverlayPosition.BOTTOM_CENTER,
    z_order=100,
))

# Dynamic update at runtime
overlay.update_overlay("title", content="Live Meeting — Recording")
overlay.remove_overlay("subtitle")
```

## Image overlays

```python
from roomkit.video.pipeline.overlay import ImageOverlayRenderer

overlay = OverlayFilter(renderers=[
    TextOverlayRenderer(),
    ImageOverlayRenderer(),
])

# Add a logo
with open("logo.png", "rb") as f:
    logo_bytes = f.read()

overlay.add_overlay(Overlay(
    id="logo",
    content=logo_bytes,
    overlay_type="image",
    position=OverlayPosition.TOP_LEFT,
    style={"width": 80},  # scale to 80px wide, preserve aspect ratio
))
```

## Rich overlays (tables, styled text)

Requires `pip install roomkit[video-overlay]` (Pillow).

```python
import json
from roomkit.video.pipeline.overlay import RichOverlayRenderer

overlay = OverlayFilter(renderers=[
    TextOverlayRenderer(),
    RichOverlayRenderer(),
])

# Table overlay
table = json.dumps({
    "headers": ["Metric", "Value"],
    "rows": [["Users", "42"], ["Latency", "12ms"]],
})
overlay.add_overlay(Overlay(
    id="stats",
    content=table,
    overlay_type="rich",
    position=OverlayPosition.TOP_RIGHT,
    style={"width": 250, "font_size": 14},
))
```

## Positions

| Position | Placement |
|----------|-----------|
| `TOP_LEFT` | Top-left with padding |
| `TOP_CENTER` | Top-center |
| `TOP_RIGHT` | Top-right |
| `CENTER_LEFT` | Middle-left |
| `CENTER` | Dead center |
| `CENTER_RIGHT` | Middle-right |
| `BOTTOM_LEFT` | Bottom-left |
| `BOTTOM_CENTER` | Bottom-center (default for subtitles) |
| `BOTTOM_RIGHT` | Bottom-right |
| `CUSTOM` | Manual `x`, `y` coordinates |

## Performance

- **Cached rendering**: Renderers cache their output patches. On unchanged content, they blit the cached patch — no re-rendering at 30fps.
- **Empty fast-path**: When no overlays are configured, `filter()` returns the frame immediately (no copy).
- **Thread-safe**: `OverlayFilter` uses a `threading.Lock` — safe to update overlays from async hooks while the video thread renders.

## Renderers

| Renderer | Content type | Dependencies |
|----------|-------------|-------------|
| `TextOverlayRenderer` | Text strings | `opencv`, `numpy` (via `roomkit[local-video]`) |
| `ImageOverlayRenderer` | PNG/JPEG bytes | `opencv`, `numpy` |
| `RichOverlayRenderer` | Styled text, JSON tables | `Pillow` (via `roomkit[video-overlay]`) |
| `MockOverlayRenderer` | Any (pass-through) | None |

Custom renderers: implement `OverlayRenderer` ABC with `overlay_type`, `render()`, `invalidate_cache()`, and `clear_cache()`.
