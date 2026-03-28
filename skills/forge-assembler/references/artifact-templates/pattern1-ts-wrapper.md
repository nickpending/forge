# Pattern 1: Tool Template

Use this template when generating deterministic tools for Tier 3 plans. Tools handle mechanical operations with structured JSON I/O. Zero AI — same input always produces same output.

**Default to single-file Bun.** Escalate to multi-file only when complexity demands it.

---

## Complexity Decision

| Signal | Single file (default) | Multi-file (escalation) |
|--------|----------------------|------------------------|
| Lines of code | < ~500 | > 500 or growing |
| External consumers | None — CLI only | Other packages import as library |
| Dependencies | Zero or few (Bun built-ins) | npm packages needed |
| Build pipeline | None — `bun run` directly | Bundling, publishing, type-checking |
| Maintenance | One team, one project | Shared across projects |

If ALL signals point to single-file, use the single-file template. If ANY signal points to multi-file, consider escalating — but default to single-file and refactor later if needed.

---

## Single-File Template (Default)

One `.ts` file with shebang, types, logic, and CLI entry point. No package.json, no tsconfig, no build step.

```typescript
#!/usr/bin/env bun

/**
 * {tool-name} — {one-line description}
 *
 * Deterministic tool. Same input always produces same output.
 * JSON to stdout, diagnostics to stderr.
 */

// --- Types ---

interface {ToolName}Input {
  target: string;
  // {additional input fields}
}

interface {ToolName}Result {
  status: "success" | "error";
  target: string;
  data: {ToolName}Data[];
  metadata: {
    tool: string;
    version: string;
    timestamp: string;
    duration_ms: number;
  };
}

interface {ToolName}Data {
  // {structured output fields}
}

// --- Core Logic ---

async function execute(input: {ToolName}Input): Promise<{ToolName}Result> {
  const start = Date.now();

  // {deterministic operation — tool invocation, data processing, API call}

  return {
    status: "success",
    target: input.target,
    data: [],
    metadata: {
      tool: "{tool-name}",
      version: "1.0.0",
      timestamp: new Date().toISOString(),
      duration_ms: Date.now() - start,
    },
  };
}

function validate(input: {ToolName}Input): void {
  if (!input.target) throw new Error("target is required");
  // {additional validation}
}

// --- CLI Entry ---

if (import.meta.main) {
  const args = process.argv.slice(2);

  if (args.length === 0 || args.includes("--help")) {
    console.error("Usage: {tool-name} <target> [options]");
    console.error("");
    console.error("Options:");
    console.error("  --help     Show this help message");
    process.exit(2);
  }

  const input: {ToolName}Input = {
    target: args[0],
    // {parse additional args}
  };

  try {
    validate(input);
    const result = await execute(input);
    console.log(JSON.stringify(result, null, 2));
    process.exit(0);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    console.error(`Error: ${message}`);
    console.log(JSON.stringify({ status: "error", error: message }));
    process.exit(1);
  }
}

// Export for programmatic use (optional — import.meta.main guard keeps CLI from firing)
export { execute, validate };
export type { {ToolName}Input, {ToolName}Result, {ToolName}Data };
```

### Single-File Directory Structure

```
{tool-name}/
└── {tool-name}.ts    # Everything in one file
```

Run: `bun {tool-name}.ts <args>`
Install: Kit copies to `~/.local/bin/{tool-name}`, shebang handles execution.

---

## Multi-File Template (Escalation)

Use when the tool exceeds ~500 lines, has npm dependencies, needs to be imported as a library by other packages, or requires a build pipeline.

### index.ts (Pure Library)

```typescript
/**
 * {tool-name} — {one-line description}
 *
 * Pure library. Types and exported functions only.
 * No process.exit, no stdin/stdout, no CLI concerns.
 */

// --- Types ---

export interface {ToolName}Input {
  target: string;
}

export interface {ToolName}Result {
  status: "success" | "error";
  // ...
}

// --- Core Functions ---

export async function execute(input: {ToolName}Input): Promise<{ToolName}Result> {
  // ...
}
```

### cli.ts (Thin Shell)

```typescript
#!/usr/bin/env bun

import { execute, type {ToolName}Input } from "./index.ts";

async function main(): Promise<void> {
  const args = process.argv.slice(2);
  // arg parsing, call execute(), JSON to stdout
}

main();
```

### package.json

```json
{
  "name": "@voidwire/{tool-name}",
  "version": "1.0.0",
  "type": "module",
  "exports": { ".": "./index.ts" },
  "bin": { "{tool-name}": "./cli.ts" },
  "scripts": {
    "build": "bun build ./cli.ts --outdir ./dist --target bun",
    "typecheck": "tsc --noEmit",
    "test": "bun test"
  }
}
```

### Multi-File Directory Structure

```
{tool-name}/
├── index.ts       # Pure library
├── cli.ts         # Thin shell
├── package.json   # Dual exports (library + CLI)
└── tsconfig.json  # TypeScript config
```

---

## Kit Registration

Register via CLI after committing to forge-armory:

```bash
kit add --name {tool-name} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path tools/{tool-name} \
  --type tool \
  --domain security \
  --tags {domain-tag},{tool-tag},campaign:{slug} \
  --description "One-line description of what this tool does"
```

---

## Rules

- **TypeScript only** — no bash tools (composition rule 8)
- **Zero AI** — deterministic, same input always produces same output
- **JSON output to stdout** — structured, machine-readable
- **Diagnostics to stderr** — human-readable error messages
- **Exit codes:** 0 = success, 1 = error, 2 = usage
- **Default to single-file** — escalate to multi-file only when complexity demands it
- **Bun-native** — use `#!/usr/bin/env bun` shebang, `import.meta.main` guard, no build step needed for single-file

---

## Checklist

- [ ] Has `#!/usr/bin/env bun` shebang
- [ ] Outputs JSON to stdout
- [ ] Outputs diagnostics to stderr
- [ ] Exit codes: 0 success, 1 error, 2 usage
- [ ] No AI, no reasoning, no judgment
- [ ] Same input always produces same output
- [ ] Single-file unless complexity signals warrant multi-file
- [ ] Kit registration uses --type tool
- [ ] Types exported for programmatic use (even in single-file)
