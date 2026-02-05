# AI Assistant Integration

RoomKit provides documentation optimized for AI coding assistants, helping developers get accurate, working code when using tools like Claude Code, Cursor, GitHub Copilot, and others.

## llms.txt

The `/llms.txt` file provides a structured index of RoomKit documentation following the [llms.txt specification](https://llmstxt.org/).

**Location:** `https://your-docs-site/llms.txt`

AI assistants can use this file to:

- Quickly find relevant documentation
- Understand the project structure
- Navigate to specific API references

### Format

```markdown
# RoomKit

> RoomKit is a pure async Python library for building multi-channel conversation systems...

## Getting Started

- [Features Overview](docs/features.md): Core features, channels, hooks...

## Core API

- [RoomKit](docs/api/roomkit.md): Central orchestrator class...
```

## AGENTS.md

The `AGENTS.md` file in the repository root provides project-specific guidance for AI coding assistants.

**Location:** Repository root (`/AGENTS.md`)

### What It Contains

| Section | Purpose |
|---------|---------|
| **Quick Reference** | Dev commands (uv sync, pytest, ruff) |
| **Project Structure** | Directory tree with descriptions |
| **Architecture Patterns** | ABC + Default, Channel, Provider, Hook patterns |
| **Code Style** | Required patterns, type hints, imports |
| **Testing Patterns** | Test class structure, fixtures |
| **Common Tasks** | Adding providers, channels, hooks |
| **Boundaries** | Always do / Ask first / Never do rules |
| **Key Concepts** | Room lifecycle, event flow, identity resolution |

### Why It Matters

Without `AGENTS.md`:

- AI assistants guess patterns from training data
- Code may not follow RoomKit's async-first architecture
- Missing exports, wrong patterns, inconsistent style

With `AGENTS.md`:

- AI follows RoomKit's actual patterns
- Correct ABC + default implementation pattern
- Proper hook usage (BEFORE_BROADCAST vs AFTER_BROADCAST)
- Knows to export public classes from `__init__.py`

## Usage with AI Assistants

### Claude Code / Cursor

These tools automatically read `AGENTS.md` from the repository root when working on your project.

### Explicit Reference

You can also explicitly instruct AI assistants:

```
Read AGENTS.md and implement a new RCS channel for RoomKit following the existing patterns.
```

### MCP Server (Future)

A Model Context Protocol server for live documentation search is planned. This will allow AI assistants to:

- Search RoomKit documentation in real-time
- Get up-to-date API signatures
- Find code examples across the repository

## Best Practices for Contributors

When contributing to RoomKit:

1. **Follow AGENTS.md patterns** - AI assistants will expect these patterns
2. **Update documentation** - Keep llms.txt and API docs in sync
3. **Export public classes** - Add to `roomkit/__init__.py`
4. **Add tests** - Follow the testing patterns in AGENTS.md

## Resources

- [llms.txt Specification](https://llmstxt.org/)
- [AGENTS.md Format](https://agents.md/)
- [GitHub: How to write a great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
