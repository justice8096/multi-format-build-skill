# Graphify Setup

[Graphify](https://github.com/safishamsi/graphify) turns this repo (code, docs,
schemas, papers, even images/video) into a queryable knowledge graph. It
generates an interactive `graph.html`, a `GRAPH_REPORT.md`, and a `graph.json`,
then lets you ask questions like *"what connects the manifest to the build
template?"* instead of grepping around.

These generated outputs are git-ignored (see `.gitignore`) — they're
machine-specific and meant to be rebuilt locally.

## Install (Windows, PowerShell)

```powershell
# 1. Install uv (Python tool manager)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# --- reopen PowerShell so `uv` is on PATH ---

# 2. Install graphify (PyPI package is `graphifyy` — double-y; CLI is `graphify`)
uv tool install graphifyy

# 3. Register the skill and wire it into Claude Code
graphify install
graphify claude install
```

> **WSL2 / macOS / Linux:** replace step 1 with
> `curl -LsSf https://astral.sh/uv/install.sh | sh`, then run steps 2–3 the same way.

## Usage

Open this repo in Claude Code, then:

```
# Build the graph
/graphify .

# Query it
/graphify query "what connects the manifest to the build template?"

# (optional) auto-rebuild the graph on every commit
graphify hook install
```

## Requirements

- Python 3.10+ (`uv` fetches a compatible interpreter automatically)
- `uv` (recommended) or `pipx`
- Code parsing runs locally via tree-sitter (no API calls). Semantic
  extraction of docs/PDFs reuses your Claude Code session — no separate API key
  needed inside the IDE.

## Notes

- Run `graphify claude install` *before* expecting the `/graphify` slash command
  to appear — it also tells Claude Code to consult `GRAPH_REPORT.md` for
  architecture questions.
- If `graphify` isn't recognized after install, reopen the terminal or run
  `uv tool update-shell` to refresh PATH.
