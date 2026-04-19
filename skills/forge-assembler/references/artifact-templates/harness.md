# Harness Template

Use this template when generating standalone Agent SDK orchestrator projects. Harnesses are TypeScript or Python projects with LLM-driven control flow — a root agent dynamically decides what runs next.

---

## Defaults

| Dimension | Default | Override via |
|-----------|---------|-------------|
| Language | TypeScript | plan frontmatter |
| Package manager (TS) | bun | plan frontmatter |
| Package manager (Python) | uv | plan frontmatter |
| TS verification | `bun run tsc --noEmit` | — |
| Python verification | `python -m py_compile src/main.py` + `python -c "import main"` | — |

---

## Plan-Schema Fields (Harness-Specific)

The plan carries three harness-specific fields:

| Field | Type | Description |
|-------|------|-------------|
| `harness_agents` | list[string] | Agent slugs the harness orchestrates. Each gets Kit-resolved or built in the same assembly batch. |
| `harness_schedule` | string or null | Cron expression if the harness runs on a schedule; `null` if user-launched. |
| `harness_env` | list[string] | Env var names the harness requires. Assembler emits matching entries in `.env.example`. |

---

## TypeScript Project Layout

```
{harness-name}/
├── src/
│   └── index.ts          # Entrypoint — Agent SDK query/session setup
├── .claude/
│   ├── agents/           # Embedded agent personas (from harness_agents)
│   │   └── {agent-slug}.md
│   ├── commands/         # Embedded commands (optional)
│   └── settings.json     # Harness-local Claude settings (optional)
├── package.json
├── tsconfig.json
├── .env.example          # Generated from harness_env
├── .gitignore
└── README.md             # Usage, schedule setup, env vars
```

### package.json

```json
{
  "name": "{harness-name}",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "bun run src/index.ts",
    "typecheck": "tsc --noEmit",
    "dev": "bun --watch src/index.ts"
  },
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "latest"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "declaration": true,
    "noEmit": true
  },
  "include": ["src/**/*.ts"]
}
```

### src/index.ts (Skeleton)

```typescript
import { Agent } from "@anthropic-ai/claude-agent-sdk";

/**
 * {harness-name} — {one-line description}
 *
 * LLM-driven orchestrator. Root agent dynamically decides
 * what runs next based on input/context.
 */

async function main() {
  const agent = new Agent({
    // System prompt defining the root agent's orchestration logic
    systemPrompt: `You are the orchestrator for {harness-name}.
Your job: {describe what the root agent decides and orchestrates}.

Available agents: {list from harness_agents}

Route work to the appropriate agent based on {describe routing criteria}.`,
    // Tools, MCP servers, permissions configured here
  });

  // Query mode (single task) or session mode (long-running)
  const result = await agent.query({
    prompt: process.argv[2] || "Start the pipeline.",
  });

  console.log(result);
}

main().catch(console.error);
```

---

## Python Project Layout

```
{harness-name}/
├── src/
│   └── main.py           # Entrypoint — Agent SDK setup
├── .claude/
│   ├── agents/           # Embedded agent personas
│   └── settings.json     # Harness-local settings (optional)
├── pyproject.toml
├── .env.example
├── .gitignore
└── README.md
```

### pyproject.toml

```toml
[project]
name = "{harness-name}"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = ["claude-agent-sdk"]

[project.scripts]
start = "python src/main.py"
```

### Verification

```bash
python -m py_compile src/main.py    # Syntax check
python -c "import main"              # Import smoke test
```

---

## Distinguishing from Command

| Attribute | Command | Harness |
|-----------|---------|---------|
| Runs in | Claude Code (human session) | Standalone (cron, user-launched, event-triggered) |
| Control flow | LLM via Skill() chain | LLM via Agent SDK query() |
| Practitioner present | Yes | No |
| Artifact shape | Single .md file | Full project (package.json + src/ + .claude/) |
| Kit type | `command` | `harness` |

**Forcing question:** Does the orchestrator run inside Claude Code with a practitioner present? If yes → command. If no → harness.

---

## Distinguishing from Automation Config

| Attribute | Harness | Automation Config |
|-----------|---------|-------------------|
| Control flow | LLM-driven (root agent decides next step) | Deterministic (config file decides) |
| LLM role | Control flow driver | Leaf-stage transform only |
| Artifact shape | Project with Agent SDK | Config file (Justfile, cron, n8n JSON) |
| Kit-eligible | Yes | No (armory-only) |

**Forcing question:** Does an LLM decide what happens next? If yes → harness. If no → automation config.

---

## Forge Runtime

Harnesses have their own forge runtime preamble (they are standalone programs):

1. **Check forge runtime:** Verify `~/.config/forge/context.json` exists — if not, exit with error
2. **Read context.json:** Parse for environment awareness
3. **Ensure data directory:** `mkdir -p ~/.local/share/forge/{name}/`
4. **On completion:** Write JSONL line to `~/.local/share/forge/{name}/ledger.jsonl`

---

## Kit Registration

Register via CLI after committing to forge-armory:

```bash
kit add --name {harness-name} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path harnesses/{harness-name} \
  --type harness \
  --domain security \
  --tags {domain-tag},{workflow-tag},orchestrator,campaign:{harness-name} \
  --description "One-line description of what this harness orchestrates"
```

### Install semantics

`kit use {harness-name}` clones the harness project to the user's workspace dir (e.g. `~/forge-harnesses/{harness-name}/`) and runs `bun install` (TS) or `uv sync` (Python). Launch is explicit — the user runs `bun run start` or adds it to cron themselves. Kit handles distribution; the user owns runtime management.

---

## Inner Artifacts

Agents embedded in `.claude/agents/` follow the agent-persona template. Their reusability classification determines Kit registration:

- **Reusable** (general-purpose agent usable elsewhere) → Kit-registered independently, no bundle tag
- **Parent-scoped** (designed for this harness but theoretically portable) → Kit-registered with `bundle:harness:{harness-name}` tag
- **Private** (only makes sense inside this harness) → stays in `.claude/agents/`, not Kit-registered

The assembler applies the reusability rubric per inner component.

---

## Checklist

- [ ] Project has `package.json` (TS) or `pyproject.toml` (Python) with Agent SDK dependency
- [ ] Entrypoint exists at `src/index.ts` (TS) or `src/main.py` (Python)
- [ ] `.env.example` lists all env vars from `harness_env`
- [ ] `.claude/agents/` contains persona files for each agent in `harness_agents`
- [ ] Root agent system prompt describes orchestration logic
- [ ] Verification passes: `bun run tsc --noEmit` (TS) or `python -m py_compile` (Python)
- [ ] Kit registration uses --type harness
- [ ] Inner artifacts classified per reusability rubric
- [ ] Distinguishing characteristics documented (vs command and vs automation config)
