# Local LLM with llama.cpp

`LlamaCppAIProvider` runs an open-weights model on your own machine, with nothing to install or start beside your application. You name a model; RoomKit downloads the [llama.cpp](https://github.com/ggml-org/llama.cpp) build that fits the machine (CUDA, Metal or CPU), downloads the model, starts `llama-server` on a local port, and stops it when the provider closes.

Tool calling works out of the box: the server renders each model's own chat template (`--jinja`), so tools reach the model in the format it was trained on, and RoomKit reads the calls back through the same OpenAI-compatible path it uses for vLLM.

## Install

```bash
pip install roomkit[llamacpp]
```

That is the only step. No Ollama, no Docker, no compiler.

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.llamacpp import LlamaCppAIProvider, LlamaCppConfig

provider = LlamaCppAIProvider(
    LlamaCppConfig(model="unsloth/Qwen3-4B-Instruct-2507-GGUF:Q4_K_M")
)

kit = RoomKit()
kit.register_channel(AIChannel("ai", provider=provider, system_prompt="You are helpful."))
```

`model` is the only required setting. It is either a Hugging Face reference `repo:quant` (a GGUF repository and the quantization to take from it), or the path of a local `.gguf` file.

Call `await provider.start()` at startup if you do not want the first user message to wait for the model to load. `await kit.close()` (or `await provider.close()`) stops the server.

A runnable version with two tools: [`examples/llamacpp_tools.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/llamacpp_tools.py).

## What happens on the first run

| Step | Where it goes | Typical time |
|---|---|---|
| Download the llama.cpp build for this machine, check its SHA-256 | `~/.cache/roomkit/llama.cpp/<build>/<variant>` | a few seconds to a minute |
| Download the model (`llama-server -hf`) | the Hugging Face cache, `~/.cache/huggingface/hub` | depends on the model: ~2.5 GB for a 4B Q4 |
| Load the model, answer `/health` | GPU memory, or RAM | ~20 s for a 4B model |

The next runs reuse both caches and start in seconds. Every download is logged at `INFO` on the `roomkit.providers.llamacpp` logger; the server's own output is logged at `DEBUG` on the same logger.

!!! note "Pinned and verified"
    RoomKit downloads one exact llama.cpp release, pinned in the package with the SHA-256 of every archive. A download whose checksum does not match is refused and nothing from it is extracted or run. Upgrading RoomKit is how the pinned build moves forward.

## Choosing a model

Pick an **instruct** model, not a thinking one, unless you want its reasoning: a model that always reasons spends seconds before its first word, which a voice assistant cannot afford.

Measured with Qwen3-4B-Instruct-2507 (Q4_K_M) on an RTX 4070 (12 GB) and an i7-13700KF, answering five questions with three real tools, results read back from the tools each time:

| Hardware | Choosing the tool | Final answer | Memory |
|---|---|---|---|
| GPU (all layers) | 0.2–0.5 s | 1.4–3.6 s | ~4 GB VRAM |
| CPU only (`gpu_layers=0`) | 3–7 s | 7–54 s (long tool results) | system RAM, no VRAM |

Every tool call's arguments matched the tool's JSON Schema on both. On the CPU the model chooses tools just as well; it is the long answers that are slow, so keep the CPU for text, and a GPU for voice.

Any GGUF repository works: `unsloth/…-GGUF`, `bartowski/…-GGUF`, `Qwen/…-GGUF`. Smaller quantizations (`Q4_K_M`) fit more easily; larger ones (`Q8_0`) follow instructions and tools a little better.

## Tools

Nothing is specific to llama.cpp: define tools on the `AIChannel` as with any provider.

```python
from roomkit.providers.ai.base import AITool

weather = AITool(
    name="get_weather",
    description="Current weather for a city",
    parameters={"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]},
)

async def run_tool(name: str, arguments: dict) -> str:
    return '{"temperature": -5, "sky": "snow"}'

ai = AIChannel("ai", provider=provider, tools=[weather], tool_handler=run_tool)
```

Tools from an MCP server plug in the same way through the [MCP tool provider](mcp-tool-provider.md).

## GPU or CPU

By default llama.cpp puts as many layers on the GPU as it holds and the rest on the CPU. To choose:

```python
LlamaCppConfig(model="…", gpu_layers=0)    # CPU only
LlamaCppConfig(model="…", gpu_layers=20)   # 20 layers on the GPU, the rest on the CPU
```

The build is picked from the machine:

| Machine | Build |
|---|---|
| Linux x64 with an NVIDIA driver for CUDA 13 / 12 | `linux-x64-cuda-13` / `linux-x64-cuda-12` |
| Linux x64 without NVIDIA | `linux-x64-cpu` |
| Linux arm64 (with or without CUDA 13) | `linux-arm64-cuda-13` / `linux-arm64-cpu` |
| macOS Apple silicon / Intel | `macos-arm64` (Metal) / `macos-x64` |
| Windows x64 (CUDA 13 / 12 / none), Windows arm64 | `windows-x64-cuda-13`, `windows-x64-cuda-12`, `windows-x64-cpu`, `windows-arm64-cpu` |

Force one with `variant="linux-x64-vulkan"` (AMD and Intel GPUs on Linux) or any other name from the table.

## Using your own llama-server

To run a `llama-server` you built or installed yourself:

```python
LlamaCppConfig(model="…", binary="/opt/llama.cpp/build/bin/llama-server")
LlamaCppConfig(model="…", binary="llama-server")   # looked up on the PATH
```

A `llama-server` that merely happens to be on the `PATH` is never used unless `binary` names it: an old build, a CPU-only package or a broken wrapper would otherwise decide what your application runs.

Anything else `llama-server` accepts goes through `extra_args`:

```python
LlamaCppConfig(model="…", extra_args=["--threads", "8"])
```

## Configuration

| Field | Default | What it does |
|---|---|---|
| `model` | required | `repo:quant` on Hugging Face, or a `.gguf` path |
| `context_size` | `8192` | Context window in tokens; tool definitions and results count against it |
| `gpu_layers` | `None` | Layers on the GPU; `None` = as many as fit, `0` = CPU only |
| `max_tokens` | `1024` | Maximum tokens in one answer |
| `temperature` | `0.7` | Sampling temperature |
| `enable_thinking` | `None` | Turn a reasoning model's thinking on/off through its template |
| `binary` | `None` | Your own `llama-server` (path or name on the `PATH`) |
| `variant` | `None` | Force a download variant (see the table above) |
| `cache_dir` | `~/.cache/roomkit/llama.cpp` | Where llama.cpp builds are kept |
| `port` | `None` | Local port; `None` picks a free one |
| `startup_timeout` | `1800` | Seconds to wait for the server, model download included |
| `timeout` | `120` | Seconds for one request |
| `extra_args` | `[]` | More `llama-server` arguments |

## In a voice assistant

[`examples/voice_local_vui.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_local_vui.py) runs microphone, speech recognition, this provider and Vui TTS on one machine. Vui (~3.5 GB) and Qwen3-4B Q4 (~4 GB) share a 12 GB GPU; the model's reply starts 0.4–1.4 s after the user's words are transcribed.

## Troubleshooting

**The first start takes minutes.** The model is downloading: the server reports its progress on the `roomkit.providers.llamacpp` logger at `DEBUG`. Raise `startup_timeout` for a large model on a slow link.

**`llama-server exited with code N`.** The error carries the last lines the server printed: a model reference that does not exist, a file that is not a GGUF, not enough memory. Fix what it says; to see the whole output, set the `roomkit.providers.llamacpp` logger to `DEBUG`.

**It runs on the CPU although there is a GPU.** Check `nvidia-smi` shows a driver: the CUDA build is chosen from the CUDA version the driver reports. Then force `variant="linux-x64-cuda-13"` (or 12) if detection picked the CPU build.

**Out of GPU memory.** Lower `gpu_layers`, choose a smaller quantization, or a smaller `context_size`.

**A `llama-server` process is left after a crash.** The provider stops the server on `close()` and when Python exits normally; a process killed with `SIGKILL` cannot clean up. `pkill -x llama-server` removes it.

## Maintainers: moving the pinned build

```bash
make update-llamacpp            # the newest llama.cpp build
uv run python scripts/update_llamacpp_build.py b11160   # or a given one
```

It rewrites `src/roomkit/providers/llamacpp/_builds.py` with every variant's archive and SHA-256 from the GitHub release. Run the provider tests, then try the new build on a GPU machine before releasing.
