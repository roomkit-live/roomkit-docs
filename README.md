# RoomKit Documentation

Technical documentation for [RoomKit](https://github.com/roomkit-live/roomkit), built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Local Development

```bash
pip install mkdocs mkdocs-material mkdocstrings[python]
mkdocs serve
```

Then open http://localhost:8000.

## Structure

```
docs/
  index.md            Getting started
  architecture.md     Architecture overview
  features.md         Feature reference
  technical.md        Technical deep dive
  ai-integration.md   AI channel guide
  mcp.md              MCP integration
  faq.md              FAQ
  cpaas-comparison.md CPaaS comparison
  api/                API reference (auto-generated from docstrings)
mkdocs.yml            MkDocs configuration
```

## Build

```bash
mkdocs build
```

Output goes to `site/`.

## Related Repos

- [roomkit](https://github.com/roomkit-live/roomkit) — Python library
- [roomkit-website](https://github.com/roomkit-live/roomkit-website) — Landing page
- [roomkit-specs](https://github.com/roomkit-live/roomkit-specs) — Protocol specs
