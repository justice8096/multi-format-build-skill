# Build Template Reference

This is the canonical `build.ts` template. It is **fully manifest-driven** â€” there are NO placeholder
tokens to replace. Every project-specific value (name, slug, icon, descriptions) is read dynamically
from `source/manifest.json` at build time.

**CRITICAL: Domain-agnostic rule.** The generated build.ts must NEVER contain hardcoded domain
strings. No project names, no domain keywords, no skill-specific terminology. Everything comes
from the manifest. This applies to ALL format generators including the CLI generator â€” the CLI
should construct display strings from `manifest.metadata.name` at runtime, not inline them as
string literals in the build script.

Copy this template verbatim and save as `build.ts` in the project root. Do not modify it to
include project-specific strings.

---

```typescript
import { readFileSync, writeFileSync, mkdirSync, rmSync, existsSync } from "fs";
import { resolve, dirname } from "path";
import { fileURLToPath } from "url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// ============================================================================
// TYPE DEFINITIONS
// ============================================================================

interface ParameterProperty {
  type: string;
  description: string;
  default?: string | boolean | number;
  items?: { type: string };
}

interface CommandParameters {
  type: string;
  properties: Record<string, ParameterProperty>;
  required: string[];
}

interface Command {
  name: string;
  displayName: string;
  description: string;
  path: string;
  parameters: CommandParameters;
  keywords: string[];
}

interface Skill {
  name: string;
  displayName: string;
  description: string;
  path: string;
  keywords: string[];
}

interface Template {
  name: string;
  displayName: string;
  description: string;
  path: string;
}

interface Metadata {
  name: string;
  version: string;
  description: string;
  author: string;
  license: string;
  repository?: string;
  keywords: string[];
}

interface Manifest {
  metadata: Metadata;
  skills: Skill[];
  commands: Command[];
  templates: Template[];
}

type FormatName = "claude-plugin" | "openai" | "n8n" | "prompts" | "mcp" | "cli";

// ============================================================================
// CLI ARGUMENT PARSING
// ============================================================================

interface CLIArgs {
  format: FormatName | "all";
  watch: boolean;
  clean: boolean;
  validateOnly: boolean;
}

function parseArgs(): CLIArgs {
  const args = process.argv.slice(2);
  const result: CLIArgs = { format: "all", watch: false, clean: false, validateOnly: false };
  for (let i = 0; i < args.length; i++) {
    const arg = args[i];
    if (arg === "--format" && args[i + 1]) { result.format = args[++i] as FormatName | "all"; }
    else if (arg.startsWith("--format=")) { result.format = arg.split("=")[1] as FormatName | "all"; }
    else if (arg === "--watch") { result.watch = true; }
    else if (arg === "--clean") { result.clean = true; }
    else if (arg === "--validate-only") { result.validateOnly = true; }
  }
  return result;
}

// ============================================================================
// SIMPLE YAML STRINGIFIER (zero dependencies)
// ============================================================================

function stringifyYaml(obj: unknown, indent: number = 2): string {
  function stringify(val: unknown, depth: number = 0): string {
    const spaces = " ".repeat(depth * indent);
    if (val === null || val === undefined) return "null";
    if (typeof val === "string") {
      if (val.includes("\n") || val.includes(":") || val.includes("#")) {
        return "'" + val.replace(/'/g, "''") + "'";
      }
      return val;
    }
    if (typeof val === "number" || typeof val === "boolean") return String(val);
    if (Array.isArray(val)) {
      if (val.length === 0) return "[]";
      const items: string[] = [];
      for (const item of val) { items.push("- " + stringify(item, depth + 1)); }
      return items.join("\n");
    }
    if (typeof val === "object") {
      const entries = Object.entries(val);
      if (entries.length === 0) return "{}";
      const items: string[] = [];
      for (const [key, value] of entries) {
        if (typeof value === "object" && value !== null) {
          items.push(key + ":");
          const nested = stringify(value, depth + 1);
          for (const line of nested.split("\n")) { items.push("  " + line); }
        } else { items.push(key + ": " + stringify(value, depth + 1)); }
      }
      return items.join("\n");
    }
    return String(val);
  }
  return stringify(obj);
}

// ============================================================================
// UTILITY FUNCTIONS
// ============================================================================

function ensureDir(dirPath: string): void { mkdirSync(dirPath, { recursive: true }); }

function readMarkdown(filePath: string): string {
  try {
    let content = readFileSync(filePath, "utf-8");
    if (content.charCodeAt(0) === 0xfeff) { content = content.slice(1); }
    return content;
  } catch { console.warn("Warning: Could not read " + filePath); return ""; }
}

function log(message: string): void { console.log("[build] " + message); }
function logSuccess(message: string): void { console.log("\u2713 " + message); }
function logError(message: string): void { console.error("\u2717 " + message); }

function toSlug(name: string): string {
  return name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "");
}
function toPascalCase(name: string): string {
  return name.split(/[-_\s]+/).map((w) => w.charAt(0).toUpperCase() + w.slice(1)).join("");
}
function toSnakeCase(name: string): string { return name.replace(/-/g, "_"); }

// ============================================================================
// LOAD AND VALIDATE
// ============================================================================

function loadManifest(): Manifest {
  log("Loading manifest.json...");
  const manifestPath = resolve(__dirname, "source/manifest.json");
  const raw = JSON.parse(readFileSync(manifestPath, "utf-8"));
  if (!raw.metadata || !raw.metadata.name || !raw.metadata.version || !raw.metadata.description || !raw.metadata.author || !raw.metadata.license || !Array.isArray(raw.metadata.keywords)) { throw new Error("Manifest missing required metadata fields (name, version, description, author, license, keywords)");
  }
  if (!Array.isArray(raw.commands) || raw.commands.length === 0) {
    throw new Error("Manifest must have at least one command");
  }
  if (!Array.isArray(raw.skills)) { throw new Error("Manifest must have a skills array"); }
  if (!Array.isArray(raw.templates)) { raw.templates = []; }
  for (const cmd of raw.commands) {
    if (!cmd.parameters || !cmd.parameters.properties) {
      throw new Error(`Command "${cmd.name}" missing JSON Schema parameters`);
    }
  }
  logSuccess("Manifest loaded (" + raw.commands.length + " commands, " + raw.skills.length + " skills)");
  return raw as Manifest;
}

function loadSkillMarkdown(skillName: string): string {
  return readMarkdown(resolve(__dirname, "source/skills/" + skillName + ".md"));
}
function loadCommandMarkdown(commandName: string): string {
  return readMarkdown(resolve(__dirname, "source/commands/" + commandName + ".md"));
}

// ============================================================================
// FORMAT 1: CLAUDE PLUGIN
// ============================================================================

function generateClaudePlugin(manifest: Manifest): void {
  log("Generating Claude Code plugin...");
  const base = resolve(__dirname, "dist/claude-plugin");
  ensureDir(base); ensureDir(resolve(base, "skills")); ensureDir(resolve(base, "commands"));
  const slug = toSlug(manifest.metadata.name);
  const plugin = {
    name: slug, version: manifest.metadata.version, description: manifest.metadata.description,
    author: manifest.metadata.author, license: manifest.metadata.license,
    keywords: manifest.metadata.keywords,
    skills: manifest.skills.map((s) => ({ name: s.name, description: s.description, path: "skills/" + s.name })),
    commands: manifest.commands.map((c) => ({ name: c.name, description: c.description, path: "commands/" + c.name + ".md" })),
  };
  writeFileSync(resolve(base, "plugin.json"), JSON.stringify(plugin, null, 2));
  for (const skill of manifest.skills) {
    ensureDir(resolve(base, "skills/" + skill.name));
    writeFileSync(resolve(base, "skills/" + skill.name + "/SKILL.md"), loadSkillMarkdown(skill.name));
  }
  for (const cmd of manifest.commands) {
    writeFileSync(resolve(base, "commands/" + cmd.name + ".md"), loadCommandMarkdown(cmd.name));
  }
  logSuccess("Claude plugin generated");
}

// ============================================================================
// FORMAT 2: OPENAI FUNCTIONS
// ============================================================================

function generateOpenAIFunctions(manifest: Manifest): void {
  log("Generating OpenAI function schemas...");
  ensureDir(resolve(__dirname, "dist/openai"));
  const functions = manifest.commands.map((cmd) => {
    const properties: Record<string, unknown> = {};
    const required: string[] = cmd.parameters.required || [];
    for (const [paramName, paramDef] of Object.entries(cmd.parameters.properties)) {
      const prop: Record<string, unknown> = { type: paramDef.type === "array" ? "array" : paramDef.type, description: paramDef.description };
      if (paramDef.items) prop.items = paramDef.items;
      if (paramDef.default !== undefined) prop.default = paramDef.default;
      properties[paramName] = prop;
    }
    return { type: "function", function: { name: toSnakeCase(cmd.name), description: cmd.description, parameters: { type: "object", properties, required } } };
  });
  writeFileSync(resolve(__dirname, "dist/openai/functions.json"), JSON.stringify(functions, null, 2));
  logSuccess("OpenAI functions generated");
}

// ============================================================================
// FORMAT 3: N8N NODE
// ============================================================================

function generateN8nNode(manifest: Manifest): void {
  log("Generating n8n node definition...");
  ensureDir(resolve(__dirname, "dist/n8n"));
  const pascalName = toPascalCase(manifest.metadata.name);
  const slug = toSlug(manifest.metadata.name);
  const operations = manifest.commands.map((cmd) => ({ name: cmd.displayName, value: toSnakeCase(cmd.name), description: cmd.description }));
  const paramFields = manifest.commands.flatMap((cmd) =>
    Object.entries(cmd.parameters.properties).map(([paramName, paramDef]) => ({
      displayName: paramName.replace(/_/g, " ").replace(/\b\w/g, (c) => c.toUpperCase()),
      name: paramName,
      type: paramDef.type === "array" ? "string" : paramDef.type === "boolean" ? "boolean" : "string",
      required: (cmd.parameters.required || []).includes(paramName),
      default: paramDef.default ?? "", description: paramDef.description,
      displayOptions: { show: { operation: [toSnakeCase(cmd.name)] } },
    }))
  );
  const node = {
    displayName: manifest.metadata.name, name: slug.replace(/-/g, ""), icon: "file:icon.svg",
    group: ["transform"], version: 1, description: manifest.metadata.description,
    defaults: { name: manifest.metadata.name }, inputs: ["main"], outputs: ["main"],
    properties: [{ displayName: "Operation", name: "operation", type: "options", options: operations, default: operations[0]?.value || "" }, ...paramFields],
  };
  writeFileSync(resolve(__dirname, "dist/n8n/" + pascalName + ".node.json"), JSON.stringify(node, null, 2));
  logSuccess("n8n node definition generated");
}

// ============================================================================
// FORMAT 4: PROMPT LIBRARY
// ============================================================================

function generatePromptLibrary(manifest: Manifest): void {
  log("Generating prompt library...");
  ensureDir(resolve(__dirname, "dist/prompts"));
  const promptIndex = {
    name: manifest.metadata.name + " Prompts", version: manifest.metadata.version,
    description: manifest.metadata.description, author: manifest.metadata.author,
    prompts: manifest.commands.map((cmd) => ({ id: cmd.name, name: cmd.displayName, description: cmd.description, file: cmd.name + ".yaml" })),
  };
  writeFileSync(resolve(__dirname, "dist/prompts/index.yaml"), stringifyYaml(promptIndex, 2));
  const expertise = manifest.skills.map((s) => s.description);
  for (const cmd of manifest.commands) {
    const prompt = {
      name: cmd.displayName, id: cmd.name, description: cmd.description,
      models: ["gpt-4", "gpt-4-turbo", "claude-3-opus", "claude-3-sonnet", "claude-3.5-sonnet"],
      context: { role: "You are an expert assistant for " + manifest.metadata.name + ". " + manifest.metadata.description, expertise },
      parameters: Object.entries(cmd.parameters.properties).map(([name, def]) => ({
        name, type: def.type, required: (cmd.parameters.required || []).includes(name), description: def.description, default: def.default,
      })),
      output: { format: "markdown", description: "Comprehensive " + cmd.displayName.toLowerCase() + " document" },
    };
    writeFileSync(resolve(__dirname, "dist/prompts/" + cmd.name + ".yaml"), stringifyYaml(prompt, 2));
  }
  logSuccess("Prompt library generated");
}

// ============================================================================
// FORMAT 5: MCP SERVER
// ============================================================================

function generateMcpServer(manifest: Manifest): void {
  log("Generating MCP server...");
  const base = resolve(__dirname, "dist/mcp-server");
  ensureDir(resolve(base, "src/tools")); ensureDir(resolve(base, "src/knowledge"));
  ensureDir(resolve(base, "knowledge/skills")); ensureDir(resolve(base, "knowledge/commands"));
  const slug = toSlug(manifest.metadata.name);

  for (const cmd of manifest.commands) {
    const toolName = toSnakeCase(cmd.name);
    const requiredParams = cmd.parameters.required || [];
    const zodFields: string[] = []; const jsonSchemaProps: string[] = [];
    for (const [paramName, paramDef] of Object.entries(cmd.parameters.properties)) {
      let zField = "  " + paramName + ": z.";
      if (paramDef.type === "array") zField += "array(z.string())";
      else if (paramDef.type === "boolean") zField += "boolean()";
      else if (paramDef.type === "number") zField += "number()";
      else zField += "string()";
      if (!requiredParams.includes(paramName)) zField += ".optional()";
      zField += ","; zodFields.push(zField);
      let jsProp = "      " + paramName + ': { type: "' + paramDef.type + '", description: "' + cmd.description.replace(/"/g, '\\"') + '" }';
      jsonSchemaProps.push(jsProp);
    }
    const toolCode = ['import { z } from "zod";', 'import { loadSkillContent } from "../knowledge/loader.js";', "",
      "const " + toolName + "Schema = z.object({", ...zodFields, "});", "",
      "export const " + toolName + 'Definition = { name: "' + toolName + '", description: "' + cmd.description.replace(/"/g, '\\"') + '",',
      "  inputSchema: { type: \"object\" as const, properties: {", ...jsonSchemaProps.map((p) => p + ","),
      "    }, required: [" + requiredParams.map((p) => '"' + p + '"').join(", ") + "] } };", "",
      "export async function handle(input: Record<string, unknown>): Promise<string> {",
      "  const validated = " + toolName + "Schema.parse(input);",
      '  const skillContent = await loadSkillContent("' + cmd.name + '");',
      '  return JSON.stringify({ status: "success", command: "' + cmd.name + '", message: "Tool executed.", skillPreview: skillContent.slice(0, 200), input: validated }, null, 2);',
      "}"].join("\n");
    writeFileSync(resolve(base, "src/tools/" + toolName + ".ts"), toolCode);
  }

  const loaderCode = ['import { readFileSync } from "fs";', 'import { resolve, dirname } from "path";', 'import { fileURLToPath } from "url";', "",
    "const __filename = fileURLToPath(import.meta.url);", "const __dirname = dirname(__filename);", "",
    "const CACHE = new Map<string, string>();", "",
    "export async function loadSkillContent(skillId: string): Promise<string> {",
    '  if (CACHE.has(skillId)) return CACHE.get(skillId)!;', "  try {",
    '    const p = resolve(__dirname, "../../knowledge/skills/" + skillId + ".md");',
    '    const content = readFileSync(p, "utf-8");', "    CACHE.set(skillId, content); return content;",
    '  } catch { return ""; }', "}", "",
    "export async function loadCommandContent(commandId: string): Promise<string> {",
    '  try { return readFileSync(resolve(__dirname, "../../knowledge/commands/" + commandId + ".md"), "utf-8"); }',
    '  catch { return ""; }', "}"].join("\n");
  writeFileSync(resolve(base, "src/knowledge/loader.ts"), loaderCode);

  const toolImports: string[] = []; const toolDefs: string[] = []; const toolHandlerCases: string[] = [];
  for (const cmd of manifest.commands) {
    const toolName = toSnakeCase(cmd.name);
    toolImports.push("import { " + toolName + "Definition, handle as handle_" + toolName + ' } from "./tools/' + toolName + '.js";');
    toolDefs.push("  " + toolName + "Definition,");
    toolHandlerCases.push('      case "' + toolName + '": return { content: [{ type: "text", text: await handle_' + toolName + "(args) }] };");
  }
  const indexCode = ['import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";',
    'import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";', "",
    ...toolImports, "", "const tools = [", ...toolDefs, "];", "",
    "async function main(): Promise<void> {",
    '  const server = new McpServer({ name: "' + slug + '-mcp", version: "' + manifest.metadata.version + '" });',
    "  const transport = new StdioServerTransport();",
    '  server.server.setRequestHandler("tools/list", async () => ({ tools }));',
    '  server.server.setRequestHandler("tools/call", async (request: { params: { name: string; arguments?: Record<string, unknown> } }) => {',
    "    const { name, arguments: args = {} } = request.params;", "    switch (name) {",
    ...toolHandlerCases, '      default: throw new Error("Unknown tool: " + name);', "    }", "  });",
    "  await server.connect(transport);",
    '  console.error("' + manifest.metadata.name + ' MCP Server running on stdio");', "}", "",
    'main().catch((error) => { console.error("Fatal error:", error); process.exit(1); });'].join("\n");
  writeFileSync(resolve(base, "src/index.ts"), indexCode);

  const mcpPkg = { name: "@" + slug + "/mcp-server", version: manifest.metadata.version,
    description: manifest.metadata.description, author: manifest.metadata.author,
    license: manifest.metadata.license, type: "module", main: "src/index.ts",
    scripts: { start: "tsx src/index.ts", build: "tsc", "type-check": "tsc --noEmit" },
    dependencies: { "@modelcontextprotocol/sdk": "^0.6.0", zod: "^3.22.4" },
    devDependencies: { "@types/node": "^20.10.0", typescript: "^5.3.3", tsx: "^4.7.0" } };
  writeFileSync(resolve(base, "package.json"), JSON.stringify(mcpPkg, null, 2));

  const mcpTsconfig = { compilerOptions: { target: "ES2020", module: "ESNext", lib: ["ES2020"],
    moduleResolution: "bundler", strict: true, esModuleInterop: true, skipLibCheck: true,
    forceConsistentCasingInFileNames: true, resolveJsonModule: true, declaration: true,
    outDir: "./dist", rootDir: "./src" }, include: ["src/**/*"], exclude: ["node_modules"] };
  writeFileSync(resolve(base, "tsconfig.json"), JSON.stringify(mcpTsconfig, null, 2));

  for (const skill of manifest.skills) {
    writeFileSync(resolve(base, "knowledge/skills/" + skill.name + ".md"), loadSkillMarkdown(skill.name));
  }
  for (const cmd of manifest.commands) {
    writeFileSync(resolve(base, "knowledge/commands/" + cmd.name + ".md"), loadCommandMarkdown(cmd.name));
  }
  logSuccess("MCP server generated");
}

// ============================================================================
// FORMAT 6: STANDALONE CLI
// ============================================================================

function generateCli(manifest: Manifest): void {
  log("Generating standalone CLI...");
  const base = resolve(__dirname, "dist/cli"); ensureDir(base);
  const commandHelp = manifest.commands.map((cmd) => '  "  ' + cmd.name + ' -- ' + cmd.description + '\\n" +').join("\n");
  const commandCases: string[] = [];
  for (const cmd of manifest.commands) {
    const paramLines: string[] = [];
    for (const [paramName, paramDef] of Object.entries(cmd.parameters.properties)) {
      const isRequired = (cmd.parameters.required || []).includes(paramName);
      paramLines.push("        " + paramName + ': args["--' + paramName.replace(/_/g, "-") + '"] || ' +
        (paramDef.default !== undefined ? JSON.stringify(paramDef.default) :
          isRequired ? '(() => { console.error("Missing required: --' + paramName.replace(/_/g, "-") + '"); process.exit(1); })()' : "undefined") + ",");
    }
    commandCases.push(['    case "' + cmd.name + '":', "      return {", '        command: "' + cmd.name + '",',
      '        displayName: "' + cmd.displayName + '",', "        params: {", ...paramLines, "        },", "      };"].join("\n"));
  }
  const cliCode = ["#!/usr/bin/env tsx",
    "// Auto-generated CLI â€” project name is read from manifest at build time",
    "// Run: tsx cli.ts <command> [--param value ...]", "",
    'const VERSION = "' + manifest.metadata.version + '";',
    'const NAME = "' + manifest.metadata.name + '";', "",
    "function showHelp(): void {",
    '  console.log(NAME + " CLI v" + VERSION);', '  console.log("");',
    '  console.log("Usage: tsx cli.ts <command> [options]");', '  console.log("");',
    '  console.log("Commands:");', "  console.log(", commandHelp, '  ""', "  );",
    '  console.log("Options:");', '  console.log("  --help     Show this help message");',
    '  console.log("  --version  Show version");', "}", "",
    "function parseCliArgs(): Record<string, string> {",
    "  const args: Record<string, string> = {};", "  const raw = process.argv.slice(3);",
    "  for (let i = 0; i < raw.length; i++) {",
    '    if (raw[i].startsWith("--") && raw[i + 1] && !raw[i + 1].startsWith("--")) { args[raw[i]] = raw[++i]; }',
    '    else if (raw[i].includes("=")) { const [key, ...val] = raw[i].split("="); args[key] = val.join("="); }',
    "  }", "  return args;", "}", "",
    "function routeCommand(command: string, args: Record<string, string>): unknown {",
    "  switch (command) {", ...commandCases, "    default:",
    '      console.error("Unknown command: " + command);', "      showHelp();", "      process.exit(1);",
    "  }", "}", "", "const command = process.argv[2];", "",
    'if (!command || command === "--help") { showHelp(); process.exit(0); }',
    'if (command === "--version") { console.log(VERSION); process.exit(0); }', "",
    "const args = parseCliArgs();", "const result = routeCommand(command, args);",
    "console.log(JSON.stringify(result, null, 2));"].join("\n");
  writeFileSync(resolve(base, "cli.ts"), cliCode);
  logSuccess("Standalone CLI generated");
}

// ============================================================================
// FORMAT REGISTRY
// ============================================================================

const FORMAT_GENERATORS: Record<FormatName, (manifest: Manifest) => void> = {
  "claude-plugin": generateClaudePlugin, openai: generateOpenAIFunctions,
  n8n: generateN8nNode, prompts: generatePromptLibrary, mcp: generateMcpServer, cli: generateCli,
};

// ============================================================================
// MAIN BUILD
// ============================================================================

async function main(): Promise<void> {
  const args = parseArgs();
  try {
    if (args.clean) {
      log("Cleaning dist/...");
      if (existsSync(resolve(__dirname, "dist"))) { rmSync(resolve(__dirname, "dist"), { recursive: true }); }
      logSuccess("Cleaned");
      if (args.format === "all" && !args.validateOnly) { /* continue */ }
      else if (args.validateOnly) { /* continue */ }
      else { return; }
    }
    log("Starting build..."); log("Output directory: " + resolve(__dirname, "dist"));
    const manifest = loadManifest();
    if (args.validateOnly) { logSuccess("Manifest is valid!"); return; }
    if (args.format === "all") {
      for (const [name, generator] of Object.entries(FORMAT_GENERATORS)) { generator(manifest); }
    } else {
      const generator = FORMAT_GENERATORS[args.format];
      if (!generator) throw new Error("Unknown format: " + args.format);
      generator(manifest);
    }
    log(""); logSuccess("Build completed successfully!"); log("Generated artifacts:");
    if (args.format === "all" || args.format === "claude-plugin") log("  - dist/claude-plugin/ (plugin.json + skills/ + commands/)");
    if (args.format === "all" || args.format === "openai") log("  - dist/openai/functions.json");
    if (args.format === "all" || args.format === "n8n") log("  - dist/n8n/*.node.json");
    if (args.format === "all" || args.format === "prompts") log("  - dist/prompts/*.yaml");
    if (args.format === "all" || args.format === "mcp") log("  - dist/mcp-server/ (full TypeScript MCP server)");
    if (args.format === "all" || args.format === "cli") log("  - dist/cli/cli.ts (standalone CLI)");
  } catch (error) {
    logError("Build failed!");
    if (error instanceof Error) console.error(error.message); else console.error(error);
    process.exit(1);
  }
}

main().catch((err) => { console.error("Build failed:", err); process.exit(1); });
```
