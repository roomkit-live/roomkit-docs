# Skills Served over MCP

An MCP server can publish [Agent Skills](agent-skills.md) beside its tools, under the MCP Skills extension (`io.modelcontextprotocol/skills`, [SEP-2640](https://modelcontextprotocol.io/seps/2640-skills-extension)). RoomKit is a host for it: `SkillRegistry.add_mcp_server()` registers the skills a server publishes in the same registry as the host's own, and every channel that serves the registry serves them too (RFC §24.7).

A served skill is someone else's content. RoomKit reads it lazily, checks it against what the server listed, sets it apart from the host's own instructions, and never runs code on its behalf without the host's approval.

```bash
pip install roomkit[mcp]
```

---

## Quick start

```python
from roomkit import AIChannel
from roomkit.skills import SkillRegistry
from roomkit.tools import MCPToolProvider

async with MCPToolProvider.from_url("https://huggingface.co/mcp", discover=False) as hf:
    registry = SkillRegistry()
    registry.discover("./skills")              # the host's own skills, unchanged
    count = await registry.add_mcp_server(hf, "hf")
    print(registry.skill_names)                # [..., "hf:hf-cli", "hf:huggingface-zerogpu", ...]

    ai = AIChannel("ai-assistant", provider=provider, skills=registry)
```

`discover=False` connects without listing the server's tools: the skills are what this host wants from it. A server that does not declare the extension raises `SkillDiscoveryError` rather than registering nothing in silence.

The model sees the served skills in its manifest, each naming its origin, its description quoted as text from outside, and activates one with `activate_skill` as it would a host skill:

```xml
<available_skills>
  <skill name="hf:hf-cli" origin="hf">
    <description>“Hugging Face Hub CLI (`hf`) for downloading, uploading, and managing …”</description>
  </skill>
</available_skills>
```

---

## Names

Each skill is registered as `<label>:<path>`. The label is the host's name for the server, an identifier the host chooses, never the name the server gives itself (any server can claim any name). The path is the skill's URI without `/SKILL.md`, and without the scheme when it is `skill://`:

| Entry URI | Name under label `hf` |
|-----------|-----------------------|
| `skill://hf-cli/SKILL.md` | `hf:hf-cli` |
| `skill://acme/billing/refunds/SKILL.md` | `hf:acme/billing/refunds` |
| `github://owner/repo/skills/refunds/SKILL.md` | `hf:github://owner/repo/skills/refunds` |

A host skill's name never holds `:` (`registry.add()` refuses one), so a served skill never shadows a host's skill, and two servers serving the same URI are two skills. A label already held by another server is refused.

---

## What is read, and when

| Step | What RoomKit reads |
|------|--------------------|
| `add_mcp_server()` | `skills/list` only: every page, no file |
| `activate_skill(name)` | the skill's `SKILL.md`, from its own server |
| `read_skill_reference(name, path)` | that one file, if the entry lists it |

**Verification.** The listing gives each file of a skill with its size and SHA-256 digest. Before anything of a `SKILL.md` reaches the model, RoomKit checks its length, its digest and its frontmatter, parsed and compared field by field with the entry's. A mismatch loads nothing: the model is told the skill changed on its server or could not be verified, never its content, and the entry is read again (`skills/get`) for the next activation.

**Files.** `read_skill_reference` takes any path the entry lists (`references/forms.md`, `templates/invoice.md`), not only `references/`. A path the entry does not list is refused unread, whatever the server would serve, and a read never reaches another server. A file that fails verification retires the skill: the model is told to activate it again, which delivers its current version.

**Where the body goes.** A served skill's body follows the [activation lifecycle](agent-skills.md#activation-lifecycle) of any skill, with one difference: it is the server's data, not the host's rules. It rides the system prompt under its own heading, after the host's skills, in a `<skill>` block naming the label and the URI:

```text
# Active skills served by MCP servers

## Skill: hf:huggingface-zerogpu (served by MCP server hf, skill://huggingface-zerogpu/SKILL.md)
<skill>
...
</skill>
```

An activation is not replayed from the stored history: after a restart, its next activation fetches and verifies the body again.

---

## Unavailable entries

An entry RoomKit does not take is registered as **unavailable** with its reason, so the model can say why instead of guessing; the server's other skills stay usable:

- a malformed entry (no `resources`, a `name` that is not the URI's last segment, a file outside the skill's directory, a file listed twice);
- an entry past 512 files or 16 MiB in all;
- a **dynamic** entry (`resources: "dynamic"`, whose content the server generates and nothing can verify), unless the host allows them:

```python
await registry.add_mcp_server(server, "live", allow_dynamic=True)
```

A dynamic skill is read unverified, within 16 MiB in all.

An entry read again after a failed check is held to the same rules: a server that answers with a dynamic entry where it listed a verified one leaves the skill unavailable.

---

## Skills the listing does not hold

A listing may be partial. An activation naming `<label>:<path>` that the registry does not hold, under a label it holds, asks the server with `skills/get`. A skill it returns is registered unlisted (activatable, absent from the manifest) and activated; an error answers that the server serves no such skill. This is how a skill a server's own instructions point to is reached.

---

## Registering again, and copies

Calling `add_mcp_server()` again for the same server replaces its entries. A skill it no longer offers, or offers in a form the host does not take, retires; one it still offers keeps, while active, the entry it was loaded from. It clears the marks of those names, as registering a directory again does.

A registry's copies share the servers' entries:

```python
agent_skills = registry.copy(["hf:hf-cli", "code-review"])   # this agent's subset
```

- A skill retired by registering the server again is retired in every copy, an active one included.
- A copy narrowed to some names reaches those names only: it never asks the server for another one.
- A copy made with `marks=False` never asks the server for a name its source marked unavailable.

---

## Running code

Nothing a served skill says runs code on its own:

- its `allowed_tools` frontmatter grants nothing and opens nothing;
- `run_skill_script` refuses it, since its files are never written to disk to be run;
- while it is active in a conversation, a call to a tool that runs code is refused unless the host approves.

The tools that run code are the sandbox's commands and every tool matching the `execution_tools` patterns (globs). The host approves through `approve_execution`:

```python
from roomkit.skills import SkillExecutionRequest

reviewed = {"sha256:9f2c…"}   # fingerprints a person reviewed

async def approve(request: SkillExecutionRequest) -> bool:
    # request.name, .label, .uri, .fingerprint, .room_id, .tool
    return request.label == "internal" and request.fingerprint in reviewed

await registry.add_mcp_server(
    server,
    "internal",
    approve_execution=approve,
    execution_tools=["run_*", "deploy_service"],
)
```

- Every served skill active in the conversation must be approved; without a callback the call is always refused, and the model reads the refusal.
- An approval holds for the conversation and the entry's **fingerprint** (the SHA-256 of its files' URIs and digests): an entry read again with other files is asked again. A dynamic skill has no fingerprint and is asked every time.
- A callback that raises, or answers anything but `True`, refuses.

---

## Realtime voice

A [`RealtimeVoiceChannel`](realtime-voice-providers.md) serves them the same way in `skill_delivery_mode="on_demand"`: the manifest names the origin, `activate_skill` fetches and verifies the body, and a fixed provider receives it in the tool result, fenced the same way. A session keeps an activated body as it rendered it.

In `inline_full` mode, every body is handed over when the session starts, which would fetch every served skill ahead of need: there, served skills are unavailable, with that reason.

---

## What the connection exposes

`MCPToolProvider` carries what the registry needs, and a host can use it directly:

| Member | Description |
|--------|-------------|
| `server_extensions` | The extensions the server declared when it connected (`{"io.modelcontextprotocol/skills": {...}}`) |
| `request(method, params=None, timeout=30)` | Sends a raw JSON-RPC request and returns its result as a dict |
| `read_resource(uri)` | Reads a resource (a skill's file) |

The extension works over the session `mcp` 1.x negotiates (protocol `2025-06-18` or `2025-11-25`): a listing without the newer `ttlMs`, `cacheScope` and `resultType` fields is read as complete.

---

## Conformance

`tests/test_mcp_skills_conformance.py` runs the host scenarios of the [MCP conformance suite](https://github.com/modelcontextprotocol/conformance) against the suite's own content and answers: no file read on connection or listing (`sep-2640-client-no-prefetch`), and a `SKILL.md` whose digest, size or frontmatter does not match its entry refused, its supporting file never read (`sep-2640-client-verify-digest`, `-verify-size`, `-verify-frontmatter`).

Not supported yet: `resources/directory/read` for dynamic skills, and keeping fetched content on disk between reads (content is kept in memory for the entry held).

---

## Example

[`examples/mcp_skills.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/mcp_skills.py) registers the skills huggingface.co/mcp serves, loads one and reads one of its files (no API key needed), then, with `--chat`, opens a CLI conversation with an agent that activates them:

```bash
uv run python examples/mcp_skills.py
ANTHROPIC_API_KEY=sk-... uv run python examples/mcp_skills.py --chat
```
