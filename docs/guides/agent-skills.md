# Agent Skills

RoomKit supports the [Agent Skills](https://agentskills.io) open standard for packaging knowledge, instructions, and scripts into reusable skill bundles that AI channels can activate at runtime. This complements MCP (runtime tool integration) with a structured knowledge-packaging format adopted by Claude Code, Cursor, Gemini CLI, VS Code, and others.

```bash
pip install roomkit
```

No extra dependencies required — skills are built into the core package.

---

## What are Agent Skills?

An Agent Skill is a directory containing a `SKILL.md` file with YAML frontmatter and a markdown body. Skills can optionally include scripts and reference files:

```
my-skill/
├── SKILL.md           # Frontmatter + instructions
├── scripts/           # Optional executable scripts
│   └── run.sh
└── references/        # Optional reference documents
    └── api-spec.md
```

The `SKILL.md` file follows this format:

```markdown
---
name: my-skill
description: A brief description of what this skill does
license: MIT
---

# Instructions

Detailed instructions for the AI on how to use this skill...
```

**Key rules:**

- `name` must be kebab-case (`^[a-z0-9]+(-[a-z0-9]+)*$`), 1-64 chars
- `name` must match the directory name
- `description` is required, max 1024 chars
- Instructions body is standard markdown

---

## Quick start

```python
from roomkit import AIChannel, SkillRegistry
from roomkit.providers.ai.mock import MockAIProvider

# 1. Discover skills from a directory
registry = SkillRegistry()
registry.discover("./skills")  # scans subdirectories for SKILL.md

# 2. Pass to AIChannel
ai = AIChannel(
    "ai-assistant",
    provider=MockAIProvider(),
    system_prompt="You are a helpful assistant.",
    skills=registry,
)

# 3. That's it — the AI now has access to:
#    - activate_skill(name)        → load full instructions
#    - read_skill_reference(...)   → read reference files
```

When the AI channel processes an event, it automatically:

1. Appends an `<available_skills>` XML block to the system prompt
2. Registers `activate_skill` and `read_skill_reference` tools
3. Intercepts skill tool calls and returns skill content

---

## SkillRegistry

The registry discovers and manages skills. It uses a two-level loading strategy:

- **Level 1 (discover/register)** — Parses frontmatter only (~100 tokens). Lightweight enough for startup.
- **Level 2 (get_skill)** — Loads full instructions on demand and caches for subsequent access.

```python
from roomkit import SkillRegistry

registry = SkillRegistry()

# Scan one or more directories
count = registry.discover("./skills", "./extra-skills")
print(f"Found {count} skills")

# Or register individually
meta = registry.register("./skills/code-review")

# Access metadata (already loaded)
for meta in registry.all_metadata():
    print(f"  {meta.name}: {meta.description}")

# Load full skill on demand (cached after first call)
skill = registry.get_skill("code-review")
if skill:
    print(skill.instructions)
    print(skill.list_scripts())
    print(skill.list_references())
```

### Prompt XML generation

The registry generates a spec-compliant `<available_skills>` XML block for injection into the system prompt:

```python
xml = registry.to_prompt_xml()
# <available_skills>
#   <skill name="code-review">
#     <description>Review code for bugs and style issues</description>
#   </skill>
#   <skill name="test-writer">
#     <description>Generate test cases from source code</description>
#   </skill>
# </available_skills>
```

Content is HTML-escaped to prevent prompt injection.

---

## AIChannel integration

Pass the registry (and optionally a script executor) to `AIChannel`:

```python
from roomkit import AIChannel, SkillRegistry

registry = SkillRegistry()
registry.discover("./skills")

ai = AIChannel(
    "ai-assistant",
    provider=provider,
    system_prompt="You are a helpful assistant.",
    skills=registry,
    # script_executor=my_executor,  # optional — see Script Execution below
)
```

### Auto-registered tools

| Tool | When available | What it does |
|------|---------------|-------------|
| `activate_skill(name)` | Always (when skills present) | Returns full instructions + lists scripts/references |
| `read_skill_reference(skill_name, filename)` | Always | Reads a file from the skill's `references/` directory |
| `run_skill_script(skill_name, script_name, arguments)` | Only when `script_executor` is set | Executes a script via the integrator's executor |

### How it works

1. **System prompt injection** — The channel appends a preamble and `<available_skills>` XML to the system prompt. If no script executor is configured, a note is added.

2. **Tool handler wrapping** — The channel wraps the user's `tool_handler` with an internal dispatcher that intercepts skill tool names and delegates everything else to the user handler.

3. **Streaming guard** — When skills are configured, streaming is disabled to allow the tool call loop. This ensures the AI can call `activate_skill` before responding.

### Combining with user tools

Skills work alongside user-defined tools from binding metadata:

```python
await kit.attach_channel("support-room", "ai-assistant",
    category=ChannelCategory.INTELLIGENCE,
    metadata={
        "tools": [
            {"name": "lookup_order", "description": "Look up an order"},
        ],
    },
)

# The AI now has: lookup_order + activate_skill + read_skill_reference
```

The user's `tool_handler` receives calls for user-defined tools; skill tools are handled internally.

---

## Script execution

Script execution is intentionally left to the integrator — there is no default implementation. This ensures you control sandboxing, timeouts, and allowed interpreters.

### Implementing a ScriptExecutor

```python
import asyncio
from roomkit import ScriptExecutor, ScriptResult, Skill

class SubprocessExecutor(ScriptExecutor):
    """Example executor using subprocess — customize for your security needs."""

    async def execute(
        self,
        skill: Skill,
        script_name: str,
        arguments: dict[str, str] | None = None,
    ) -> ScriptResult:
        script_path = skill.path / "scripts" / script_name
        if not script_path.is_file():
            return ScriptResult(exit_code=1, stderr="Script not found", success=False)

        cmd = [str(script_path)]
        if arguments:
            for k, v in arguments.items():
                cmd.extend([f"--{k}", v])

        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=str(skill.path),
        )
        stdout, stderr = await asyncio.wait_for(proc.communicate(), timeout=30)

        return ScriptResult(
            exit_code=proc.returncode or 0,
            stdout=stdout.decode(),
            stderr=stderr.decode(),
            success=proc.returncode == 0,
        )
```

### Passing to AIChannel

```python
executor = SubprocessExecutor()

ai = AIChannel(
    "ai-assistant",
    provider=provider,
    skills=registry,
    script_executor=executor,  # enables run_skill_script tool
)
```

When `script_executor` is set, the `run_skill_script` tool is added and the AI can execute scripts from skill directories.

---

## Reference files

Skills can include reference documents in a `references/` directory. The AI can read these via the `read_skill_reference` tool.

```python
# In a skill directory:
# my-skill/references/api-spec.md
# my-skill/references/schema.json

skill = registry.get_skill("my-skill")
content = skill.read_reference("api-spec.md")
```

Path traversal is blocked — filenames containing `..`, `/`, or `\` are rejected with a `ValueError`.

---

## Error handling

### Parse and validation errors

```python
from roomkit import SkillParseError, SkillValidationError, SkillRegistry

registry = SkillRegistry()

# discover() warns and skips invalid skills
count = registry.discover("./skills")  # logs warnings, returns valid count

# register() raises on invalid skills
try:
    registry.register("./skills/bad-skill")
except SkillParseError as e:
    print(f"Parse error: {e}")  # missing frontmatter, etc.
except SkillValidationError as e:
    print(f"Validation error: {e}")  # bad name, missing description, etc.
```

### Skill not found

When the AI calls `activate_skill` with an unknown name, the handler returns an error with the list of available skills so the AI can self-correct.

---

## Configuration reference

### SkillMetadata fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `str` | Yes | Kebab-case identifier (1-64 chars) |
| `description` | `str` | Yes | Brief description (1-1024 chars) |
| `license` | `str \| None` | No | License identifier (e.g. "MIT") |
| `compatibility` | `str \| None` | No | Compatibility hint |
| `allowed_tools` | `str \| None` | No | Allowed tool patterns |
| `extra_metadata` | `dict[str, str]` | No | Additional frontmatter keys |

### AIChannel constructor params

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `skills` | `SkillRegistry \| None` | `None` | Registry of available skills |
| `script_executor` | `ScriptExecutor \| None` | `None` | Executor for skill scripts |

---

## Example skill directory

```
skills/
├── code-review/
│   ├── SKILL.md
│   ├── scripts/
│   │   └── lint.sh
│   └── references/
│       └── style-guide.md
├── test-writer/
│   ├── SKILL.md
│   └── references/
│       ├── patterns.md
│       └── fixtures.md
└── deploy-helper/
    ├── SKILL.md
    └── scripts/
        ├── check-status.sh
        └── rollback.sh
```

See the [agent_skills.py example](https://github.com/roomkit-live/roomkit/blob/main/examples/agent_skills.py) for a complete runnable demo.
