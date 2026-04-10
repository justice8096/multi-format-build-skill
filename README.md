# Multi-Format Build Skill

A reusable Claude skill that generates TypeScript build systems for skill/plugin projects, compiling a single `source/manifest.json` into 6 distribution formats — with optional multi-language support.

## Output Formats

1. **Claude Plugin** — Ready-to-install Claude Code plugin
2. **OpenAI Functions** — Function calling schemas for GPT-4
3. **n8n Node** — Community node definition for n8n workflows
4. **Prompt Library** — Model-agnostic YAML prompt templates
5. **MCP Server** — Full TypeScript MCP server with tools
6. **Standalone CLI** — Command-line tool with subcommands

## How It Works

Write your domain knowledge once as markdown files with a standardized manifest, and the generated `build.ts` compiles everything into all 6 formats automatically.

## Multi-Language Support

Add `defaultLocale` and `locales` to your manifest metadata to enable multi-language builds:

```json
{
  "metadata": {
    "name": "My Skill",
    "defaultLocale": "en",
    "locales": ["en", "es", "fr"],
    ...
  }
}
```

Then provide:
- **Locale override files** (`source/locales/es.json`) — translated display strings for names and descriptions
- **Locale content directories** (`source/skills/es/`, `source/commands/es/`) — translated markdown files

The build system generates per-locale output directories (`dist/en/`, `dist/es/`, `dist/fr/`), each containing all 6 formats with localized content. Missing translations fall back to the default locale automatically.

**Backward compatible**: Projects without `metadata.locales` build exactly as before — no migration needed.

### Locale CLI Flags

```bash
npm run build                          # Build all formats, all locales
npm run build -- --locale es           # Build all formats, Spanish only
npm run build -- --locale es --format claude-plugin  # One locale, one format
npm run validate                       # Validate manifest including locale config
```

## Usage

When this skill is installed, ask Claude to:
- "Scaffold a new skill project for [domain]"
- "Add multi-format build output to my existing skill"
- "Generate a build system for my plugin"
- "Localize my skill for Spanish and French"
- "Add i18n support to my existing skill project"

## Manifest Schema

See `references/manifest-schema.md` for the full schema reference, including locale override file format.

## Build Template

See `references/build-template.md` for the canonical domain-agnostic build script with locale support.

## License

MIT
