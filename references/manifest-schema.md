# Manifest Schema Reference

The `source/manifest.json` is the single source of truth for every skill/plugin project. All 6
output formats are derived from this file.

## Full schema

```json
{
  "metadata": {
    "name": "Human-Readable Skill Name",
    "version": "1.0.0",
    "description": "One-line description of what the skill does",
    "author": "Author Name",
    "license": "MIT",
    "repository": "https://github.com/user/repo",
    "keywords": ["keyword1", "keyword2"],
    "defaultLocale": "en",
    "locales": ["en", "es", "fr"]
  },
  "skills": [
    {
      "name": "skill-slug",
      "displayName": "Human Readable Name",
      "description": "What knowledge this skill encapsulates",
      "path": "skills/skill-slug.md",
      "keywords": ["keyword1", "keyword2"]
    }
  ],
  "commands": [
    {
      "name": "command-slug",
      "displayName": "Human Readable Command Name",
      "description": "What this command does when invoked",
      "path": "commands/command-slug.md",
      "parameters": {
        "type": "object",
        "properties": {
          "param_name": {
            "type": "string",
            "description": "What this parameter controls"
          },
          "enum_param": {
            "type": "string",
            "description": "A param with fixed choices",
            "default": "option1"
          },
          "array_param": {
            "type": "array",
            "items": { "type": "string" },
            "description": "A list parameter"
          },
          "bool_param": {
            "type": "boolean",
            "description": "A flag parameter",
            "default": false
          }
        },
        "required": ["param_name"]
      },
      "keywords": ["action1", "action2"]
    }
  ],
  "templates": [
    {
      "name": "template-slug",
      "displayName": "Template Display Name",
      "description": "What this template is for",
      "path": "templates/template-slug.md"
    }
  ]
}
```

## Field reference

### metadata (required)

| Field         | Type     | Required | Description                                |
|---------------|----------|----------|--------------------------------------------|
| name          | string   | yes      | Human-readable project name                |
| version       | string   | yes      | SemVer version                             |
| description   | string   | yes      | One-line description                       |
| author        | string   | yes      | Author name                                |
| license       | string   | yes      | SPDX license identifier                    |
| repository    | string   | no       | Git repository URL                         |
| keywords      | string[] | yes      | Discovery and categorization keywords      |
| defaultLocale | string   | no       | BCP 47 locale code for the base content (default: `"en"`) |
| locales       | string[] | no       | List of all supported locale codes. When omitted, single-locale mode — build behaves exactly as before. |

### skills[] (required, may be empty)

Each skill represents a domain knowledge area loaded into context.

| Field       | Type     | Required | Description                                  |
|-------------|----------|----------|----------------------------------------------|
| name        | string   | yes      | Kebab-case slug, matches filename            |
| displayName | string   | yes      | Human-readable name                          |
| description | string   | yes      | What knowledge area this covers              |
| path        | string   | yes      | Relative path to markdown: `skills/<name>.md`|
| keywords    | string[] | yes      | Triggering keywords                          |

### commands[] (required, at least one)

Each command is a user-facing action with typed parameters.

| Field       | Type              | Required | Description                                  |
|-------------|-------------------|----------|----------------------------------------------|
| name        | string            | yes      | Kebab-case slug, matches filename            |
| displayName | string            | yes      | Human-readable name                          |
| description | string            | yes      | What the command does                        |
| path        | string            | yes      | Relative path: `commands/<name>.md`          |
| parameters  | JSONSchemaObject  | yes      | JSON Schema object with properties/required  |
| keywords    | string[]          | yes      | Discovery keywords                           |

#### parameters object

Uses standard JSON Schema `type: "object"` format:

```json
{
  "type": "object",
  "properties": {
    "param_name": {
      "type": "string | number | boolean | array",
      "description": "What this parameter does",
      "default": "optional default value",
      "items": { "type": "string" }
    }
  },
  "required": ["list", "of", "required", "param", "names"]
}
```

Supported types: `string`, `number`, `boolean`, `array` (with `items`).

### templates[] (optional)

| Field       | Type   | Required | Description                                    |
|-------------|--------|----------|------------------------------------------------|
| name        | string | yes      | Kebab-case slug                                |
| displayName | string | yes      | Human-readable name                            |
| description | string | yes      | What the template is used for                  |
| path        | string | yes      | Relative path: `templates/<name>.md`           |

---

## Locale support

### Overview

Multi-language builds are opt-in. When `metadata.locales` is present, the build system generates
per-locale output directories. When omitted, the build behaves exactly as the single-locale
version — full backward compatibility.

### Locale directory structure

Localized content lives in subdirectories named by locale code:

```
source/
├── manifest.json
├── locales/                    # Locale string overrides
│   ├── es.json
│   └── fr.json
├── skills/
│   ├── legal-framework.md      # Default locale content
│   ├── es/
│   │   └── legal-framework.md  # Spanish content
│   └── fr/
│       └── legal-framework.md  # French content
├── commands/
│   ├── run-audit.md
│   ├── es/
│   │   └── run-audit.md
│   └── fr/
│       └── run-audit.md
└── templates/
    ├── report.md
    └── es/
        └── report.md
```

### Locale override file (`source/locales/<locale>.json`)

Each locale override file contains string replacements for the manifest's human-readable fields.
Structure fields (parameters schema, paths, slugs) are NOT overridden — only display strings.

```json
{
  "metadata": {
    "name": "Nombre Legible del Skill",
    "description": "Descripción de una línea"
  },
  "skills": {
    "skill-slug": {
      "displayName": "Nombre Legible",
      "description": "Qué conocimiento abarca esta habilidad"
    }
  },
  "commands": {
    "command-slug": {
      "displayName": "Nombre del Comando",
      "description": "Lo que hace este comando",
      "parameters": {
        "param_name": {
          "description": "Descripción localizada del parámetro"
        }
      }
    }
  },
  "templates": {
    "template-slug": {
      "displayName": "Nombre de la Plantilla",
      "description": "Para qué sirve esta plantilla"
    }
  }
}
```

### Locale override field reference

| Path | Overrides | Notes |
|------|-----------|-------|
| `metadata.name` | Project display name | Used in CLI banner, plugin name, MCP server name |
| `metadata.description` | Project description | Used everywhere a description appears |
| `skills.<slug>.displayName` | Skill display name | |
| `skills.<slug>.description` | Skill description | Used in plugin.json, prompts, etc. |
| `commands.<slug>.displayName` | Command display name | Used in n8n operations, CLI help, prompts |
| `commands.<slug>.description` | Command description | Used in function schemas, tool definitions |
| `commands.<slug>.parameters.<param>.description` | Parameter description | Used in OpenAI schemas, n8n fields, CLI help |
| `templates.<slug>.displayName` | Template display name | |
| `templates.<slug>.description` | Template description | |

### Fallback chain

Content resolution follows a strict fallback chain:

1. **Locale-specific file** — `source/skills/<locale>/<name>.md`
2. **Default locale file** — `source/skills/<name>.md`

This means you can incrementally add translations. Any missing locale file falls back to the
default locale content. The same chain applies to commands and templates.

For locale override JSON files, missing fields simply retain the default locale values from
the manifest.

### What is NOT localized

These fields always stay in the default locale (typically English) regardless of build locale:

- `name` / slug fields (used as identifiers in code)
- `path` fields (filesystem paths)
- `keywords` (used for triggering/discovery — keep in default locale for consistency)
- `parameters.properties` keys (parameter names are code identifiers)
- `parameters.type`, `parameters.required` (structural schema)

---

## Migration from older formats

### Flat manifest (no metadata wrapper)

If you see top-level `name`, `version`, `description`, `author` fields, wrap them:

```json
// Before
{ "name": "...", "version": "...", "skills": [...] }

// After
{ "metadata": { "name": "...", "version": "..." }, "skills": [...] }
```

### CommandParameter[] arrays

If commands use an array of parameter objects instead of JSON Schema:

```json
// Before
"parameters": [
  { "name": "scope", "type": "enum", "required": false, "values": ["a", "b"], "default": "a", "description": "..." }
]

// After
"parameters": {
  "type": "object",
  "properties": {
    "scope": { "type": "string", "description": "...", "default": "a" }
  },
  "required": []
}
```

Note: `enum` type becomes `string` — enum values are documentation, not enforced in the schema
(they're enforced in the generated Zod schemas and n8n options instead).

### Missing fields

Add these if absent:
- `displayName` on skills/commands — derive from `name` by capitalizing and spacing
- `path` on skills/commands — derive from `name` as `skills/<name>.md` or `commands/<name>.md`
- `keywords` on skills/commands — extract key terms from `description`
- `templates` array — add empty `[]` if no templates exist

### Adding locale support to an existing single-locale project

1. Add `defaultLocale` and `locales` to `metadata`
2. Create `source/locales/` directory
3. For each new locale, create `<locale>.json` with translated display strings
4. Create locale subdirectories under `skills/`, `commands/`, `templates/`
5. Add translated markdown files in each locale subdirectory
6. Run `npm run build` — the build system auto-detects locales and generates per-locale output
