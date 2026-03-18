# Screen & Vision Tools

RoomKit provides ready-to-use **AI agent tools** that let voice or chat agents see the user's screen and webcam on demand. Instead of continuous video streaming, these tools capture a single frame when the agent needs to look — ideal for document scanning, visual Q&A, and screen assistance.

## Architecture

```
Agent decides to look
  → describe_screen / describe_webcam tool call
    → capture_screen_frame() or capture_webcam_frame()
      → VisionProvider (frame → JPEG → AI API → description)
    → result returned to agent as tool output
  → Agent answers user's question with visual context
```

## Installation

```bash
# Screen capture tools
pip install roomkit[screen-capture]    # mss (screenshot)

# Webcam capture tools
pip install roomkit[local-video]       # opencv-python-headless

# Vision providers (pick one)
pip install roomkit[openai]            # OpenAI GPT-4o
pip install roomkit[gemini]            # Gemini Vision
```

## Available Tools

| Tool | Purpose | Dependency |
|------|---------|------------|
| `DescribeScreenTool` | Capture screenshot and analyze with vision AI | `roomkit[screen-capture]` |
| `DescribeWebcamTool` | Capture webcam frame and analyze with vision AI | `roomkit[local-video]` |
| `ListWebcamsTool` | Discover available camera devices | `roomkit[local-video]` |
| `ScreenInputTools` | Keyboard/mouse control (click, type, scroll) | `roomkit[screen-input]` |

## Quick Start — Webcam Tool

A chat agent that can look through the user's webcam when asked. Tool handlers use the same `(name, args) -> str` signature everywhere — chat, voice, and video — so `.handler` methods plug in directly via `compose_tool_handlers`:

```python
import asyncio
import os

from roomkit import (
    AITool, AnthropicAIProvider, AnthropicConfig,
    ChannelCategory, DescribeWebcamTool, InboundMessage,
    ListWebcamsTool, OpenAIVisionConfig, OpenAIVisionProvider,
    RoomKit, TextContent, WebSocketChannel, compose_tool_handlers,
)
from roomkit.channels.ai import AIChannel

async def main():
    vision = OpenAIVisionProvider(OpenAIVisionConfig(
        api_key=os.environ["OPENAI_API_KEY"],
        base_url="https://api.openai.com/v1",
        model="gpt-4o",
    ))
    webcam = DescribeWebcamTool(vision, device=0)
    lister = ListWebcamsTool()

    tools = [
        AITool(
            name=webcam.definition["name"],
            description=webcam.definition["description"],
            parameters=webcam.definition["parameters"],
        ),
        AITool(
            name=lister.definition["name"],
            description=lister.definition["description"],
            parameters=lister.definition["parameters"],
        ),
    ]

    kit = RoomKit()
    ws = WebSocketChannel("ws-user")
    ai = AIChannel(
        "ai-assistant",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"]),
        ),
        system_prompt="You can see through the user's webcam. Use describe_webcam when asked to look.",
        tools=tools,
        tool_handler=compose_tool_handlers(webcam.handler, lister.handler),
    )
    kit.register_channel(ws)
    kit.register_channel(ai)

    await kit.create_room(room_id="demo")
    await kit.attach_channel("demo", "ws-user")
    await kit.attach_channel("demo", "ai-assistant", category=ChannelCategory.INTELLIGENCE)

    # Send a message — the AI will call describe_webcam
    await kit.process_inbound(InboundMessage(
        channel_id="ws-user", sender_id="user",
        content=TextContent(body="What do you see on my webcam?"),
    ))

asyncio.run(main())
```

## DescribeScreenTool

Captures a full-resolution screenshot and analyzes it with a `VisionProvider`.

```python
from roomkit import DescribeScreenTool, GeminiVisionConfig, GeminiVisionProvider

vision = GeminiVisionProvider(GeminiVisionConfig(api_key="AIza..."))
screen_tool = DescribeScreenTool(vision, monitor=1)
```

### Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `vision` | `VisionProvider` | required | Vision provider for frame analysis |
| `monitor` | `int` | `1` | Monitor index (1 = primary, 2+ = secondary) |

### Tool Schema

The tool exposes one parameter to the agent:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | `string` | yes | Visual question about the screen content |

### Usage with RealtimeVoiceChannel

```python
voice = RealtimeVoiceChannel(
    "voice",
    provider=realtime_provider,
    transport=backend,
    tools=[screen_tool.definition],
    tool_handler=screen_tool.handler,
)
```

### Programmatic Usage

```python
# Capture and analyze
description = await screen_tool.analyze("What browser tabs are open?")

# Low-level: capture only
from roomkit import capture_screen_frame
frame = capture_screen_frame(monitor=1)  # returns VideoFrame or None
```

## DescribeWebcamTool

Captures a single frame from the local webcam and analyzes it. The camera opens, discards a few warmup frames (for auto-exposure), grabs one frame, and releases immediately.

```python
from roomkit import DescribeWebcamTool, OpenAIVisionConfig, OpenAIVisionProvider

vision = OpenAIVisionProvider(OpenAIVisionConfig(
    api_key="sk-...",
    base_url="https://api.openai.com/v1",
    model="gpt-4o",
))
webcam_tool = DescribeWebcamTool(vision, device=0)
```

### Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `vision` | `VisionProvider` | required | Vision provider for frame analysis |
| `device` | `int` | `0` | Default camera device index |

### Tool Schema

The tool exposes three parameters to the agent:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | `string` | yes | Visual question about the webcam image |
| `device` | `integer` | no | Camera device index override |
| `save_path` | `string` | no | File path to save the captured frame as JPEG |

### Programmatic Usage

```python
# Analyze with default device
description = await webcam_tool.analyze("Read the text on this document")

# Analyze with specific device and save
description = await webcam_tool.analyze(
    "What is the user holding?",
    device=1,
    save_path="/tmp/capture.jpg",
)
```

### Warmup Frames

Webcams need a few frames for auto-exposure to settle. Without warmup, the first frame is often dark or over-exposed. `capture_webcam_frame` discards 5 frames by default:

```python
from roomkit import capture_webcam_frame

# Default: 5 warmup frames
frame = capture_webcam_frame(device=0)

# Skip warmup (virtual cameras that don't need it)
frame = capture_webcam_frame(device=0, warmup_frames=0)

# More warmup for slow cameras
frame = capture_webcam_frame(device=0, warmup_frames=15)
```

## ListWebcamsTool

Probes device indices and returns available cameras with their resolution.

```python
from roomkit import ListWebcamsTool

list_tool = ListWebcamsTool(max_devices=10)

# Programmatic
print(list_tool.list())
# Found 2 camera(s):
#   Device 0: Camera 0 (AVFoundation) — 1920x1080
#   Device 1: Camera 1 (AVFoundation) — 1280x720
```

### Tool Schema

No parameters — the agent calls it to discover cameras before choosing one.

### Low-level Function

```python
from roomkit import list_webcams, WebcamInfo

cameras: list[WebcamInfo] = list_webcams(max_devices=10)
for cam in cameras:
    print(f"Device {cam.device}: {cam.name} ({cam.width}x{cam.height})")
```

## Saving Frames

Both `DescribeWebcamTool` and the low-level `save_frame` function save frames as JPEG:

```python
from roomkit import capture_webcam_frame, save_frame

frame = capture_webcam_frame(device=0)
if frame:
    path = save_frame(frame, "/tmp/document_scan.jpg")
    print(f"Saved to {path}")
```

- Parent directories are created automatically
- JPEG encoding uses OpenCV (quality 85) with Pillow fallback
- If the save fails (bad path, read-only filesystem), the error is returned in the tool result so the agent can inform the user

## ScreenInputTools

For screen assistants that need to interact with the UI — click buttons, type text, scroll, and press keys:

```python
from roomkit import ScreenInputTools

input_tools = ScreenInputTools(vision=vision, monitor=1)
```

Requires `pip install roomkit[screen-input]` (`pyautogui`).

### Available Actions

| Tool | Description |
|------|-------------|
| `click_element` | Vision-assisted click — describes a UI element, vision locates it, clicks the coordinates |
| `type_text` | Type a string with optional hotkey support |
| `press_key` | Press a single key or key combination |
| `scroll` | Scroll up/down by a given amount |

### Usage with Voice Agent

```python
from roomkit import compose_tool_handlers

voice = RealtimeVoiceChannel(
    "voice",
    provider=realtime_provider,
    transport=backend,
    tools=[
        screen_tool.definition,
        *input_tools.definitions,
    ],
    tool_handler=compose_tool_handlers(
        screen_tool.handler,
        input_tools.handler,
    ),
)
```

## Combining Screen + Webcam + Input

A full screen assistant with all vision tools:

```python
from roomkit import (
    DescribeScreenTool, DescribeWebcamTool, ListWebcamsTool,
    ScreenInputTools, GeminiVisionConfig, GeminiVisionProvider,
    compose_tool_handlers,
)

vision = GeminiVisionProvider(GeminiVisionConfig(api_key="AIza..."))

screen = DescribeScreenTool(vision, monitor=1)
webcam = DescribeWebcamTool(vision, device=0)
lister = ListWebcamsTool()
inputs = ScreenInputTools(vision=vision, monitor=1)

# All tool definitions
all_tools = [
    screen.definition,
    webcam.definition,
    lister.definition,
    *inputs.definitions,
]

# Combined handler (voice context)
handler = compose_tool_handlers(
    screen.handler,
    webcam.handler,
    lister.handler,
    inputs.handler,
)
```

## Vision Providers

All screen and webcam tools accept any `VisionProvider`. Pick one based on your needs:

| Provider | Install | Best for |
|----------|---------|----------|
| `GeminiVisionProvider` | `roomkit[gemini]` | Fast cloud analysis, good OCR |
| `OpenAIVisionProvider` | `roomkit[openai]` | GPT-4o quality, broad compatibility |
| `OpenAIVisionProvider` (Ollama) | `roomkit[openai]` | Local/private, no API key needed |
| `MockVisionProvider` | built-in | Testing without API calls |

```python
# Gemini
from roomkit import GeminiVisionConfig, GeminiVisionProvider
vision = GeminiVisionProvider(GeminiVisionConfig(api_key="AIza..."))

# OpenAI GPT-4o
from roomkit import OpenAIVisionConfig, OpenAIVisionProvider
vision = OpenAIVisionProvider(OpenAIVisionConfig(
    api_key="sk-...",
    base_url="https://api.openai.com/v1",
    model="gpt-4o",
))

# Ollama (local)
vision = OpenAIVisionProvider(OpenAIVisionConfig(
    base_url="http://localhost:11434/v1",
    model="qwen3-vl:8b",
))

# Mock (testing)
from roomkit import MockVisionProvider
vision = MockVisionProvider(descriptions=["A document with text"])
```

## Examples

```bash
# Webcam assistant with Anthropic chat + OpenAI vision
ANTHROPIC_API_KEY=sk-... OPENAI_API_KEY=sk-... \
    uv run python examples/webcam_assistant.py

# Screen assistant with Gemini voice + vision
GOOGLE_API_KEY=AIza... \
    uv run python examples/screen_assistant_ia.py
```

## Use Cases

| Scenario | Tools | Description |
|----------|-------|-------------|
| **Document scanning** | `DescribeWebcamTool` | User holds document to camera, agent reads and extracts text |
| **Screen assistance** | `DescribeScreenTool` + `ScreenInputTools` | Agent sees screen, guides user, clicks buttons |
| **Object identification** | `DescribeWebcamTool` | Agent identifies physical objects shown via webcam |
| **Multi-camera monitoring** | `ListWebcamsTool` + `DescribeWebcamTool` | Agent switches between cameras as needed |
| **Visual Q&A** | `DescribeScreenTool` or `DescribeWebcamTool` | User asks questions about what's visible |
| **Accessibility** | `DescribeScreenTool` | Agent describes screen content for visually impaired users |
| **Remote support** | `DescribeScreenTool` + `ScreenInputTools` | Agent sees and controls user's desktop |
