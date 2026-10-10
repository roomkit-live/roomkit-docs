# RoomKit Documentation

Documentation site for [RoomKit](https://github.com/roomkit-live/roomkit), built
with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and
published at <https://www.roomkit.live/docs/>.

Guides, the feature reference, the architecture and the API reference live
here. The `roomkit` repository keeps only the files written for AI assistants:
`AGENTS.md`, `llms.txt`, and the topic pages in `docs/c7/` that build its
`llms-full.txt`.

## Local Development

The API reference is generated from RoomKit's docstrings, so the site builds
from a `roomkit` checkout next to this one:

```bash
cd ../roomkit
make docs-serve   # live preview on http://localhost:8000
make docs         # strict build: a broken link or an unresolved API reference fails it
```

Both targets read `../roomkit-docs` by default; pass `DOCS_DIR=<path>` to build
another checkout. The output goes to `site/` in this repository.

## Structure

```
docs/
  index.md            Home
  features.md         Feature reference
  architecture.md     Architecture overview
  technical.md        Technical details
  faq.md              FAQ
  ai-integration.md   llms.txt, AGENTS.md and Agent Skills for coding assistants
  mcp.md              MCP integration
  llms.txt            Documentation index for LLMs
  llms-full.txt       The topic pages in one file, for LLMs
  guides/             Hands-on guides, one per feature
  api/                API reference (mkdocstrings directives)
mkdocs.yml            Configuration and navigation
```

A new guide goes in `docs/guides/` and in the `nav` of `mkdocs.yml`.

## Related Repos

- [roomkit](https://github.com/roomkit-live/roomkit) — Python library
- [roomkit-website](https://github.com/roomkit-live/roomkit-website) — Landing page
- [roomkit-specs](https://github.com/roomkit-live/roomkit-specs) — Protocol specs
- [roomkit-skills](https://github.com/roomkit-live/roomkit-skills) — Agent Skills
