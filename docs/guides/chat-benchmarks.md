# Chat E2E benchmarks

The RoomKit repository includes a reusable chat benchmark in `benchmarks/chat/`.
It exercises the inbound pipeline, AIChannel, tools, skills, hooks, storage and
WebSocket delivery callbacks with synthetic conversations and explicit checks.
It runs from a source checkout; it is not installed by the RoomKit wheel.

```bash
uv run python -m benchmarks.chat --list
uv run python -m benchmarks.chat --provider mock --output benchmark-results/mock
uv run --extra cerebras python -m benchmarks.chat \
  --model qwen-3.8-27b --key-file ~/.secrets/cerebras \
  --repetitions 5 --output benchmark-results/qwen-memory
```

The initial catalog has 27 scenarios. It includes parallel and dependent tool
calls in both response modes, argument/result hooks, rejected tools, skill
activation and script execution, deferred tool discovery, four concurrent
rooms, idempotency, visibility, muting, sliding-window memory, ephemeral UI
events and injected retry/round-limit cases. `--store sqlite` exercises the same
suite with persistent storage. `--scenarios` selects a comma-separated subset.

Each run exports raw JSON, checkpointed JSONL, CSV aggregates and a Markdown
report. `--compare previous/results.json` adds a descriptive comparison. Use a
new output directory for every run. Credentials are read from the environment
or a key file and excluded from reports. Live runs consume provider tokens.

First-text latency is measured at the in-process delivery callback. Provider
wait intervals exclude downstream consumer processing; the remaining elapsed
time includes RoomKit, storage, hooks, scheduling and instrumentation. This
residual is not a CPU profile. Repeated prompts can benefit from upstream
caching, and five repetitions do not establish a production p95.

This is an extensible chat suite, not complete feature coverage. Real network
WebSocket load, MCP, HITL, orchestration, cancellation, transport resilience,
distributed backends and other advanced paths still need dedicated scenarios.
See the repository's [benchmark guide](https://github.com/roomkit-live/roomkit/blob/main/benchmarks/chat/README.md)
for the coverage matrix, methodology and extension instructions.

## Model quality

Use `--suite quality` to evaluate model correctness through the same pipeline.
Seven task families cover invoice arithmetic, constrained planning, document
precedence, conversational corrections, malicious instructions in tool data,
refund decisions and SQL aggregation. Three seeded variants per family give
21 parameterized cases. Answers are checked by deterministic oracles, not a model
judge; incorrect outputs remain available in the raw results.

```bash
uv run --extra cerebras python -m benchmarks.chat --suite quality \
  --key-file ~/.secrets/cerebras --reasoning-effort low \
  --max-tokens 4096 --repetitions 2 --output benchmark-results/quality-low
```

Compare `none` and `low` on identical seeds and variants to observe the quality
and latency tradeoff. `quality.csv` reports exact-case success, partial scores,
task correctness separately from format compliance, truncation and infrastructure
failures. Its latency statistics include
incorrect graded answers. This is a small custom evaluation, not a public
benchmark ranking or a production reliability guarantee.

See the [model-quality guide](https://github.com/roomkit-live/roomkit/blob/main/benchmarks/chat/QUALITY.md)
for rubric details, SQL isolation and reproducible comparison commands.
