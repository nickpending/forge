# Forge Runtime

The shared runtime layer that all forge-generated artifacts plug into. Defines where configuration, state, and run history live. Every generated skill, tool, agent, and command follows this contract.

---

## XDG Layout

```
~/.config/forge/                          # Configuration (user preferences + tool configs)
  context.json                            # Shared practitioner environment profile
  {thing-name}/                           # Per-tool config (created on first run)
    config.json

~/.local/share/forge/                     # Application data (state + run history)
  conops/                                 # CONOPS documents from forge-strategist
  {thing-name}/                           # Per-tool data (created on first run)
    ledger.jsonl                          # Run history — one line per completed run
    (anything else the tool needs)
```

`forge init` creates the top-level directories and `context.json`. Each tool creates its own subdirectories on first run.

---

## context.json — Shared Environment Profile

The practitioner's environment. Read by all forge artifacts for environment awareness.

**Contents:**
- Practitioner profile (skill level, preferences)
- Installed tools and versions
- Infrastructure (resolver address, automation platforms, Docker availability)
- Dataset paths
- Campaign history (added by assembler after each assembly)

**Access control:**
- **Read:** All forge-generated artifacts. Tools read context.json to understand what environment they're operating in — what resolver to use, what datasets are available, what tools are installed.
- **Write:** Only forge-strategist (updates during investigation) and forge-assembler (adds campaign entries after assembly). These run at orchestration time with full context. Generated tools run at execution time and do NOT write to context.json.

**Why read-only for tools:** If every generated tool can write to context.json, you get race conditions, conflicting updates, and no clear ownership. The strategist and assembler are the only components with the full picture.

---

## Per-Tool Configuration

Each forge-generated artifact gets its own config directory: `~/.config/forge/{name}/config.json`

**Config lifecycle:**

1. **Assembler generates defaults.** When the assembler builds an artifact, it also writes a default config file. The assembler knows the domain context from the plan — it makes informed defaults (port lists, filter signatures, resolver from context.json, output paths).

2. **First run: present and confirm.** On first invocation, the artifact checks its config. If `confirmed: false` or no `confirmed` field, it presents the config to the practitioner:
   ```
   Configuration for {name} (first run):

   {config contents, formatted readable}

   Adjust anything, or confirm to proceed.
   ```
   After confirmation, set `confirmed: true` and `confirmed_at: {ISO date}` in the config.

3. **Subsequent runs: read silently.** The artifact reads config without prompting. One-liner noting the config:
   ```
   Using config from ~/.config/forge/{name}/config.json (confirmed {date})
   ```

4. **Manual updates.** Practitioner edits the file directly or says "update config" to re-trigger the confirmation flow. Set `confirmed: false` to force re-confirmation next run.

**Shared vs tool-specific config:**
- Resolver, dataset paths, infrastructure → lives in context.json (shared). Tools READ these from context.json, never duplicate into their own config.
- Port lists, filter signatures, thresholds, output patterns → lives in per-tool config. These are specific to what THIS tool does.

---

## Run Memory (Ledger)

Each forge-generated artifact tracks its own run history: `~/.local/share/forge/{name}/ledger.jsonl`

**Format:** One JSONL line per completed run.

```jsonl
{"date":"2026-03-28T10:30:00Z","summary":"probed 2697 hosts, 1842 responded","input":"feeds/resolved/all.txt","output":"runtime/http-recon/20260328.jsonl","status":"complete","duration_ms":45200}
{"date":"2026-03-28T22:00:00Z","summary":"probed 2705 hosts, 1850 responded","input":"feeds/resolved/all.txt","output":"runtime/http-recon/20260328-2.jsonl","status":"complete","duration_ms":43100}
```

**Required fields:**
- `date` — ISO timestamp of run completion
- `summary` — one-line human-readable description of what happened
- `status` — `complete` or `error`

**Optional fields:**
- `input` — what the tool consumed
- `output` — what the tool produced (path)
- `duration_ms` — how long it took
- `counts` — tool-specific metrics (hosts probed, items filtered, etc.)

**Only completed runs get ledger entries.** If a run errors out mid-execution, write a ledger entry with `status: error` and a summary of what went wrong.

**Tools read their own ledger** to know what they've done before. "Last run was 2026-03-28, probed 2697 hosts" informs the next run's behavior (delta detection, incremental processing).

---

## Bootstrap Preamble

Every forge-generated artifact includes this preamble pattern (adapted to the artifact type):

### For Skills (Pattern 0 and Pattern 1)

```markdown
## Pre-flight: Forge Runtime

Check that the forge runtime is initialized:

\`\`\`bash
test -d ~/.config/forge && echo "FORGE_RUNTIME: OK" || echo "FORGE_RUNTIME: MISSING"
\`\`\`

If MISSING: stop and tell the practitioner: "Forge runtime not initialized. Run: `kit use forge && /forge init`"

Read the shared environment context:
\`\`\`bash
test -f ~/.config/forge/context.json && echo "CONTEXT: OK" || echo "CONTEXT: MISSING"
\`\`\`
If context exists, READ it for environment awareness (resolver, tools, dataset paths).

Check tool-specific config:
\`\`\`bash
test -f ~/.config/forge/{name}/config.json && echo "CONFIG: OK" || echo "CONFIG: MISSING"
\`\`\`
If CONFIG MISSING: create from defaults (see config defaults below) and prompt for first-run confirmation.
If CONFIG OK: read silently. Check `confirmed` field — if false, re-prompt.

Ensure data directory exists:
\`\`\`bash
mkdir -p ~/.local/share/forge/{name}
\`\`\`
```

### For Tools (Bun single-file)

The preamble is code, not markdown:

```typescript
// Forge runtime check
const forgeConfig = `${Bun.env.HOME}/.config/forge`;
const toolConfig = `${forgeConfig}/{name}/config.json`;
const toolData = `${Bun.env.HOME}/.local/share/forge/{name}`;

if (!await Bun.file(`${forgeConfig}/context.json`).exists()) {
  console.error("Forge runtime not initialized. Run: kit use forge && /forge init");
  process.exit(2);
}

// Read context for environment awareness
// Read tool config, create defaults if missing
// Ensure data directory
```

---

## Timing Profiles in Generated Artifacts

When the plan includes timing profile information (from the CONOPS), generated artifacts must support configurable timing:

1. **Config includes timing fields.** Generated config includes `timing_profile`, `rate_limit`, and `threads` with values from the plan.
2. **Skill instructions reference config.** Generated skill instructions read rate_limit and threads from config, not hardcoded values.
3. **First-run confirmation highlights timing.** When presenting config for confirmation, call out the timing profile and what it means for scan speed.

Timing profile definitions originate from the strategist's `forge-timing.md` reference and flow through the CONOPS and plan. The assembler uses the values in the plan.

---

## Assembler Responsibilities

When generating any artifact, the assembler must:

1. **Include the forge runtime preamble** appropriate to the artifact type
2. **Generate default config** at `~/.config/forge/{name}/config.json` with:
   - Sensible defaults informed by the plan's domain context
   - `confirmed: false` (triggers first-run confirmation)
   - Comments explaining each setting
3. **Reference context.json** for shared environment data — never duplicate resolver, dataset paths, or infrastructure details into per-tool config
4. **Include ledger write** at the end of each run — one JSONL line to `~/.local/share/forge/{name}/ledger.jsonl`

---

## forge init

`/forge init` creates the XDG skeleton. Idempotent — safe to run multiple times.

Creates:
```bash
mkdir -p ~/.config/forge
mkdir -p ~/.local/share/forge/conops

# context.json from sample if absent
test -f ~/.config/forge/context.json || \
  cp {context-sample.yaml} ~/.config/forge/context.json
```

Does NOT create per-tool directories — those are created by each tool on its first run.
