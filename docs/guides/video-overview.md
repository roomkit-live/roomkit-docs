# Video Support Overview

RoomKit's video subsystem brings real-time video capture, AI-powered vision, recording, and avatar rendering into the same room-based architecture used for voice and text. Video flows through the same channels, hooks, and rooms as every other modality.

---

## Architecture at a Glance

```
VideoBackend (capture/transport)
    │
    ▼
VideoChannel (session lifecycle, hooks)
    │
    ▼
VideoPipeline (optional processing)
    │
    ├─ [Decoder] → [Resizer] → [Transforms] → [Filters]
    │
    ├─ VisionProvider (frame → text description)
    │       │
    │       ▼
    │   ON_VISION_RESULT hook / inject into AI context
    │
    └─ Recorder (frames → MP4)
```

A **VideoBackend** captures or receives frames. A **VideoChannel** wires the backend into a room — managing sessions, firing hooks, and optionally running a **VideoPipeline** for decoding, resizing, filtering, and vision analysis. Results flow into the conversation via hooks or direct AI context injection.

---

## Video Backends

Backends are pure transports — they capture or receive frames and deliver them via callbacks.

| Backend | Source | Install | Guide |
|---------|--------|---------|-------|
| `LocalVideoBackend` | Webcam (OpenCV) | `roomkit[local-video]` | [Video & Vision](video-vision.md) |
| `ScreenCaptureBackend` | Screen / monitor / region | `roomkit[screen-capture]` | [Screen Capture](screen-capture.md) |
| `RTPVideoBackend` | Direct RTP endpoints | `roomkit[rtp]` | [Video Backends](video-backends.md) |
| `SIPVideoBackend` | Full SIP signaling (INVITE/BYE/SDP) | `roomkit[sip]` | [Video Backends](video-backends.md) |
| `FastRTCVideoBackend` | WebRTC via FastRTC | `roomkit[fastrtc]` | — |
| `WebSocketVideoBackend` | Browser WebSocket | — | — |
| `MockVideoBackend` | Simulated frames | Built-in | — |

RTP and SIP backends extend their voice counterparts — a single backend handles both audio and video for a call, with audio-only fallback when the remote party doesn't offer video.

Screen capture supports multi-monitor selection, region cropping, resolution downscaling (saves vision API tokens), and diff-based frame skipping for static screens.

---

## Vision Providers

Vision providers analyze video frames and return text descriptions, OCR results, and face detections.

| Provider | API | Install |
|----------|-----|---------|
| `GeminiVisionProvider` | Google Gemini | `roomkit[gemini]` |
| `OpenAIVisionProvider` | OpenAI / Ollama / vLLM | `roomkit[openai]` |
| `MockVisionProvider` | Testing | Built-in |

Vision results are delivered through the `ON_VISION_RESULT` hook and can be automatically injected into AI conversation context:

```python
# Continuous: wire vision results into an AI channel
setup_video_vision(kit, room_id="room", ai_channel_id="ai")

# Continuous: wire vision results into a realtime voice session
setup_realtime_vision(kit, room_id="room", voice_channel_id="voice")
```

For on-demand analysis (agent tools instead of continuous sampling), see [Screen & Vision Tools](screen-vision-tools.md).

---

## Agent Vision Tools

AI agents can capture and analyze frames on demand — no continuous streaming required:

| Tool | What it does |
|------|-------------|
| `DescribeScreenTool` | Capture screenshot → vision AI description |
| `DescribeWebcamTool` | Capture webcam frame → vision AI description |
| `ListWebcamsTool` | Enumerate available cameras |
| `ScreenInputTools` | Vision-assisted click, type, scroll, press keys |

These are `Tool` protocol objects — pass them directly to any channel via `tools=[...]`:

```python
from roomkit import Agent, DescribeScreenTool, ScreenInputTools

screen_tool = DescribeScreenTool(vision=gemini_vision, monitor=1)
input_tools = ScreenInputTools(vision=gemini_vision, monitor=1)

agent = Agent("assistant", provider=ai, tools=[screen_tool, *input_tools.tools])
```

See [Screen & Vision Tools](screen-vision-tools.md) for the full guide.

---

## Video Pipeline

Optional processing chain applied to inbound frames:

```
[Decoder] → [Resizer] → [Transforms] → [Filters] → taps / vision / recorder
```

| Stage | Purpose | Implementations |
|-------|---------|----------------|
| **Decoder** | Compressed → raw frames | `PyAVVideoDecoder` |
| **Resizer** | Resolution scaling | `PyAVVideoResizer` |
| **Transform** | Visual effects | Grayscale, blur, and more |
| **Filter** | Annotation / detection | `WatermarkFilter`, `YOLOFilter`, `CensorFilter` |
| **Encoder** | Raw → compressed (outbound) | `PyAVVideoEncoder` (H.264) |

```python
from roomkit import VideoPipelineConfig
from roomkit.video.pipeline.filter.watermark import WatermarkFilter

video = VideoChannel(
    "video", backend=backend,
    pipeline=VideoPipelineConfig(
        filters=[WatermarkFilter("RoomKit | {timestamp}", position="bottom-right")],
    ),
)
```

---

## Recording

Three recording layers, from per-session to room-wide:

| Layer | Scope | Output | Guide |
|-------|-------|--------|-------|
| Audio pipeline recorder | Per voice session | WAV | [WAV Recorder](wav-file-recorder.md) |
| Video pipeline recorder | Per video session | MP4 | [PyAV Video Recorder](pyav-video-recorder.md) |
| **Room media recorder** | **All channels in a room** | **Single A/V MP4** | [Room Media Recorder](room-media-recorder.md) |

The room-level recorder muxes audio and video from multiple channels into a single file with A/V sync maintained via a shared monotonic clock:

```python
from roomkit import ChannelRecordingConfig, MediaRecordingConfig, RoomRecorderBinding
from roomkit.recorder.pyav import PyAVMediaRecorder

voice = VoiceChannel("voice", ..., recording=ChannelRecordingConfig(audio=True))
video = VideoChannel("video", ..., recording=ChannelRecordingConfig(video=True))

room = await kit.create_room(
    room_id="my-room",
    recorders=[RoomRecorderBinding(
        recorder=PyAVMediaRecorder(),
        config=MediaRecordingConfig(storage="./recordings", video_codec="auto"),
    )],
)
```

`codec="auto"` uses NVIDIA NVENC when available, falls back to libx264.

---

## Avatars

Photorealistic talking-head avatars with lip-synced video output.

| Provider | Type | Guide |
|----------|------|-------|
| `AnamRealtimeProvider` | Cloud (WebRTC) | [Anam AI Avatar](anam-avatar.md) |
| `MuseTalkAvatarProvider` | Local (GPU) | — |
| `WebSocketAvatarProvider` | Custom WebSocket | — |

`RealtimeAVBridge` wires any voice/video backend to an avatar provider, handling audio resampling, H.264 encoding, video pipeline, and session lifecycle:

```python
from roomkit import AnamConfig, AnamRealtimeProvider
from roomkit.voice.realtime.bridge import RealtimeAVBridge

bridge = RealtimeAVBridge(
    AnamRealtimeProvider(AnamConfig(api_key="...", avatar_id="...")),
    sip_backend,
)
```

For room-based integration with hooks, use `RealtimeAudioVideoChannel` instead.

---

## Hook Triggers

| Hook | Type | When |
|------|------|------|
| `ON_VIDEO_SESSION_STARTED` | Async | Video session connected |
| `ON_VIDEO_SESSION_ENDED` | Async | Video session disconnected |
| `ON_VIDEO_TRACK_ADDED` | Async | New video track in session |
| `ON_VIDEO_TRACK_REMOVED` | Async | Video track removed |
| `ON_VISION_RESULT` | Async | Vision analysis completed |
| `ON_SCREEN_SHARE_STARTED` | Async | Screen share began |
| `ON_SCREEN_SHARE_STOPPED` | Async | Screen share ended |

---

## Quick Start by Use Case

| I want to... | Backend | + | Guide |
|--------------|---------|---|-------|
| **Analyze webcam frames with AI** | `LocalVideoBackend` | `GeminiVisionProvider` | [Video & Vision](video-vision.md) |
| **Build a screen assistant** | `ScreenCaptureBackend` | `ScreenInputTools` | [Screen & Vision Tools](screen-vision-tools.md) |
| **Add video to SIP calls** | `SIPVideoBackend` | — | [Video Backends](video-backends.md) |
| **Record video sessions** | Any backend | `PyAVVideoRecorder` | [PyAV Video Recorder](pyav-video-recorder.md) |
| **Record full A/V rooms** | Voice + Video | `PyAVMediaRecorder` | [Room Media Recorder](room-media-recorder.md) |
| **Show an AI avatar** | `SIPVideoBackend` | `AnamRealtimeProvider` | [Anam AI Avatar](anam-avatar.md) |
| **Stream via WebRTC** | `FastRTCVideoBackend` | — | — |

---

## Examples

| Example | What it demonstrates |
|---------|---------------------|
| `webcam_vision.py` | Webcam + Gemini vision analysis |
| `webcam_assistant.py` | Chat agent with on-demand webcam tool |
| `webcam_recording.py` | OpenCV video recording |
| `webcam_censor.py` | YOLO + censor filter pipeline |
| `screen_describe.py` | Screen capture + vision description |
| `screen_assistant_ia.py` | Voice + screen AI assistant |
| `screen_agent_orchestrated.py` | Multi-agent screen automation |
| `sip_video_call.py` | Full SIP audio+video call |
| `rtp_video_call.py` | Direct RTP video |
| `sip_anam_avatar.py` | SIP → Anam avatar bridge |
| `pyav_video_recorder.py` | H.264/NVENC recording |
| `room_media_recorder.py` | Room-level A/V muxing |
| `webrtc_video.py` | WebRTC video backend |
| `websocket_video.py` | WebSocket video transport |
