# Multi-Format Build Skill

A reusable Claude skill that generates TypeScript build systems for skill/plugin projects, compiling a single `source/manifest.json` into 6 distribution formats.

## Output Formats

1. **Claude Plugin** — Ready-to-install Claude Code plugin
2. **OpenAI Functions** — Function calling schemas for GPT-4
3. **n8n Node** — Community node definition for n8n workflows
4. **Prompt Library** — Model-agnostic YAML prompt templates
5. **MCP Server** — Full TypeScript MCP server with tools
6. **Standalone CLI** — Command-line tool with subcommands

## How It Works

Write your domain knowledge once as markdown files with a standardized manifest, and the generated `build.ts` compiles everything into all 6 formats automatically.

## Usage

When this skill is installed, ask Claude to:
- "Scaffold a new skill project for [domain]"
- "Add multi-format build output to my existing skill"
- "Generate a build system for my plugin"

## Manifest Schema

See `references/manifest-schema.md` for the full schema reference.

## Build Template

See `references/build-template.md` for the canonical domain-agnostic build script.

## License

MIT
