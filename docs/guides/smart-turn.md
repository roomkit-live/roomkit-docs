# Smart Turn Detection

Audio-native turn detection using [pipecat-ai/smart-turn](https://github.com/pipecat-ai/smart-turn). Instead of relying on silence duration to decide when a user is done speaking, the smart-turn model analyzes **prosody and intonation** from raw speech audio to distinguish mid-utterance pauses from actual turn completions.

```bash
pip install roomkit[smart-turn]
```

This installs `numpy`, `onnxruntime`, and `transformers`.

---

## Why audio-native turn detection?

Traditional turn detection in voice assistants uses one of two approaches:

- **Silence-based**: wait for N milliseconds of silence after speech ends. Simple but either cuts the user off mid-thought (short timeout) or adds noticeable latency (long timeout).
- **Text-based (LLM)**: send the transcript to an LLM to decide if the turn is complete. Accurate but adds network round-trip latency.

**Smart-turn** operates directly on the audio signal — the Whisper Tiny encoder (~8M params, ~8MB ONNX) has learned prosodic patterns that indicate turn completion:

- Falling intonation at end of a statement
- Rising intonation for questions
- Mid-sentence pauses (e.g. thinking, searching for a word) vs. final pauses
- Natural breath patterns

This gives faster and more natural conversational turn-taking without requiring text analysis.

---

## Download the model

The ONNX model files are hosted on HuggingFace. No auto-download — you must download manually:

```bash
# CPU (int8 quantized, ~8MB — recommended for most setups)
wget https://huggingface.co/pipecat-ai/smart-turn-v3/resolve/main/smart-turn-v3.2-cpu.onnx

# GPU (fp32, ~32MB — use with provider="cuda")
wget https://huggingface.co/pipecat-ai/smart-turn-v3/resolve/main/smart-turn-v3.2-gpu.onnx
```

| Model | Size | Use when |
|---|---|---|
| `smart-turn-v3.2-cpu.onnx` | 8.68 MB | CPU inference (int8 quantized, recommended) |
| `smart-turn-v3.2-gpu.onnx` | 32.4 MB | CUDA inference (fp32) |

---

## Quick start

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.turn import SmartTurnConfig, SmartTurnDetector
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADConfig, SherpaOnnxVADProvider

# VAD is required — it detects speech segments and provides audio to the turn detector
vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(model="ten-vad.onnx")
)

# Smart turn detector analyzes the audio from each speech segment
turn_detector = SmartTurnDetector(
    SmartTurnConfig(model_path="smart-turn-v3.2-cpu.onnx")
)

pipeline = AudioPipelineConfig(
    vad=vad,
    turn_detector=turn_detector,
)
```

The audio flows through the pipeline as:

```
Mic → [Pipeline] → VAD → SPEECH_END(audio) → STT → SmartTurnDetector(audio)
                                                          ↓
                                               complete? → route to AI
                                               incomplete? → wait for more speech
```

---

## Configuration

All parameters are set via `SmartTurnConfig`:

| Parameter | Default | Description |
|---|---|---|
| `model_path` | *(required)* | Path to the ONNX model file. |
| `threshold` | `0.5` | Completion probability threshold (0–1). |
| `num_threads` | `1` | CPU threads for the ONNX runtime session. |
| `provider` | `"cpu"` | ONNX execution provider (`"cpu"` or `"cuda"`). |
| `fallback_on_no_audio` | `True` | Behavior when `audio_bytes` is `None` (see below). |

---

## Tuning the threshold

The threshold controls how confident the model must be that a turn is complete before routing to the AI:

| Threshold | Behavior | Best for |
|---|---|---|
| **0.3** | Eager — completes turns quickly | Command-style interactions, noisy environments |
| **0.5** (default) | Balanced | General conversational use |
| **0.7** | Patient — waits for natural endings | Multi-sentence user responses, storytelling |

```python
# More patient — waits for clear turn endings
turn_detector = SmartTurnDetector(
    SmartTurnConfig(
        model_path="smart-turn-v3.2-cpu.onnx",
        threshold=0.7,
    )
)
```

---

## How it works

### Audio accumulation

When the turn detector says "incomplete", the audio accumulates across speech segments. On the next `SPEECH_END`, the detector sees all accumulated audio (up to 8 seconds) — not just the latest segment. This gives the model full prosodic context.

```
User speaks: "I was wondering..."     → SPEECH_END → SmartTurn: incomplete (0.3)
User pauses (1s)
User speaks: "...if you could help me" → SPEECH_END → SmartTurn: complete (0.8)
                                                        ↓
                                          Route combined text: "I was wondering if you could help me"
```

When a turn completes:

- The accumulated text from all segments is combined and routed to the AI
- The audio buffer is cleared for the next turn
- `ON_TURN_COMPLETE` hook fires with the combined text and confidence

### Inference pipeline

For each evaluation:

1. Raw int16 PCM audio from VAD is converted to float32
2. Audio is truncated to the last 8 seconds (or zero-padded if shorter)
3. Mel spectrogram features are extracted via `WhisperFeatureExtractor`
4. ONNX inference produces a single logit → sigmoid → probability
5. If probability >= threshold → turn complete

### Error handling

The detector **fails open** — if ONNX inference errors, it returns `is_complete=True` with low confidence (0.1). This ensures the conversation never gets stuck waiting.

### Lazy initialization

The ONNX session and `WhisperFeatureExtractor` are created lazily on the first `evaluate()` call, not at construction time. This means:

- Constructing `SmartTurnDetector` is fast (only validates imports and config)
- Model loading happens only when the first speech segment arrives
- The `transformers` library (heavy) is only imported at first use

---

## Hooks

SmartTurnDetector integrates with the existing turn detection hooks:

```python
from roomkit import HookExecution, HookTrigger

@kit.hook(HookTrigger.ON_TURN_COMPLETE, execution=HookExecution.ASYNC)
async def on_turn_complete(event, ctx):
    print(f"Turn complete (confidence={event.confidence:.2f}): {event.text}")

@kit.hook(HookTrigger.ON_TURN_INCOMPLETE, execution=HookExecution.ASYNC)
async def on_turn_incomplete(event, ctx):
    print(f"Waiting for more (confidence={event.confidence:.2f}): {event.text}")
```

---

## Continuous STT mode

In continuous STT mode (no local VAD — the STT provider handles endpointing), `audio_bytes` is `None` because there are no discrete speech segments. The `fallback_on_no_audio` config controls behavior:

| `fallback_on_no_audio` | Behavior |
|---|---|
| `True` (default) | Returns `is_complete=True` — acts as if no turn detector is configured |
| `False` | Returns `is_complete=False` — blocks routing (not recommended in continuous mode) |

For continuous STT, smart-turn adds no value since there's no raw audio to analyze. Leave `fallback_on_no_audio=True` (default) so it doesn't interfere.

---

## Cleanup

Call `close()` when done to release the ONNX session:

```python
turn_detector.close()
```

Or let `VoiceChannel.close()` handle the pipeline lifecycle.

---

## Full example

See [`examples/voice_smart_turn.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_smart_turn.py) for a complete working example with:

- Local mic/speakers via `LocalAudioBackend`
- sherpa-onnx STT (streaming Zipformer) + TTS (VITS/Piper) + VAD (TEN-VAD)
- Smart-turn audio-native turn detection
- Local LLM via Ollama/vLLM

The [`examples/voice_local_onnx_vllm.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_local_onnx_vllm.py) example also supports smart-turn via the `SMART_TURN_MODEL` environment variable.
