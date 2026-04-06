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
    "keywords": ["keyword1", "keyword2"]
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
