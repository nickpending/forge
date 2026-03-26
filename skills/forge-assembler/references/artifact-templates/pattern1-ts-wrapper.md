# Pattern 1: TypeScript Wrapper Template

Use this template when generating deterministic wrapper tools for Tier 3 plans. Wrappers handle mechanical operations with structured JSON I/O. Zero AI — same input always produces same output.

---

## index.ts (Pure Library)

```typescript
#!/usr/bin/env bun

/**
 * {wrapper-name} — {one-line description}
 *
 * Pure library. Types and exported functions only.
 * No process.exit, no stdin/stdout, no CLI concerns.
 */

// --- Types ---

export interface {WrapperName}Input {
  target: string;
  // {additional input fields}
}

export interface {WrapperName}Result {
  status: "success" | "error";
  target: string;
  data: {WrapperName}Data[];
  metadata: {
    tool: string;
    version: string;
    timestamp: string;
    duration_ms: number;
  };
}

export interface {WrapperName}Data {
  // {structured output fields}
}

// --- Core Functions ---

export async function execute(input: {WrapperName}Input): Promise<{WrapperName}Result> {
  const start = Date.now();

  // {deterministic operation — tool invocation, API call, data processing}

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

// --- Helpers ---

function validate(input: {WrapperName}Input): void {
  if (!input.target) {
    throw new Error("target is required");
  }
  // {additional validation}
}
```

---

## cli.ts (Thin Shell)

```typescript
#!/usr/bin/env bun

/**
 * CLI entry point for {wrapper-name}.
 * Thin shell: arg parsing, calls index.ts, JSON to stdout, diagnostics to stderr.
 */

import { execute, type {WrapperName}Input } from "./index.ts";

async function main(): Promise<void> {
  const args = process.argv.slice(2);

  if (args.length === 0 || args.includes("--help")) {
    console.error("Usage: {wrapper-name} <target> [options]");
    console.error("");
    console.error("Options:");
    console.error("  --help     Show this help message");
    process.exit(2);
  }

  const input: {WrapperName}Input = {
    target: args[0],
    // {parse additional args}
  };

  try {
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

main();
```

---

## package.json

```json
{
  "name": "@voidwire/{wrapper-name}",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./index.ts"
  },
  "bin": {
    "{wrapper-name}": "./cli.ts"
  },
  "scripts": {
    "build": "bun build ./cli.ts --outdir ./dist --target bun",
    "typecheck": "tsc --noEmit",
    "test": "bun test"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

---

## Directory Structure

```
{wrapper-name}/
├── index.ts       # Pure library — types, exported functions, no process.exit
├── cli.ts         # Thin shell — #!/usr/bin/env bun, arg parsing, JSON stdout, exit codes
├── package.json   # Dual exports (library + CLI)
└── tsconfig.json  # TypeScript config
```

---

## Kit Manifest

```yaml
name: {wrapper-name}
type: tool
repo: git@github.com:nickpending/forge-armory.git
path: tools/{wrapper-name}/
domain: security
tags: [{domain-tag}, {tool-tag}]
description: One-line description of what this wrapper does
```

---

## Rules

- **TypeScript only** — no bash tools (composition rule 8)
- **Zero AI** — deterministic, same input always produces same output
- **JSON output to stdout** — structured, machine-readable
- **Diagnostics to stderr** — human-readable error messages
- **Exit codes:** 0 = success, 1 = error, 2 = usage
- **index.ts is a pure library** — no process.exit, no stdin/stdout interaction
- **cli.ts is a thin shell** — arg parsing, calls index.ts, formats output

---

## Checklist

- [ ] index.ts exports types and functions only (no process.exit)
- [ ] cli.ts has `#!/usr/bin/env bun` shebang
- [ ] cli.ts outputs JSON to stdout
- [ ] cli.ts outputs diagnostics to stderr
- [ ] Exit codes: 0 success, 1 error, 2 usage
- [ ] package.json has dual exports (library `.` and bin)
- [ ] No AI, no reasoning, no judgment in any file
- [ ] Same input always produces same output
- [ ] kit-manifest.yaml type is `tool`
