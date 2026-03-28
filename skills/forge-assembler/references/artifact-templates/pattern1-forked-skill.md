# Pattern 1: Forked Skill-as-Agent Template

Use this template when generating Pattern 1 (forked skill-agent) artifacts for Tier 3 plans. The skill IS the agent — it runs as an autonomous subprocess with its own model and scoped tools.

---

## Skill File: SKILL.md

```markdown
---
name: {skill-slug}
description: USE WHEN {trigger conditions}. {What it does in one sentence}.
context: fork
agent: {persona-slug}
model: sonnet
allowed-tools: Read, Bash, Grep, Glob
user-invocable: false
---

# {Skill Title}

**Mission:** {One-sentence statement of what this forked agent accomplishes}

## $ARGUMENTS

This skill receives structured input from the parent:

| Argument | Type | Description |
|----------|------|-------------|
| `target` | string | {What is being operated on} |
| `scope` | string | {Boundaries of the operation} |
| `output_path` | string | {Where to write results} |

## Workflow

### Phase 1: Input Validation

1. Parse $ARGUMENTS from stdin
2. Validate required fields are present
3. Confirm target is accessible (if applicable)

### Phase 2: {Core Operation}

1. {Step with tool invocation}
2. {Step with structured output capture}
3. {Step with error handling}

### Phase 3: {Analysis / Interpretation}

1. {Step that applies judgment to Phase 2 output}
2. {Decision point: what the results mean}

### Phase 4: Output

1. Write structured results to `output_path`
2. Write operator log

## Output Format

```json
{
  "status": "complete",
  "target": "{target}",
  "findings": [
    {
      "type": "{finding-type}",
      "detail": "{description}",
      "confidence": "high|medium|low",
      "evidence": "{raw data reference}"
    }
  ],
  "summary": "{one-line summary}",
  "operator_log": "{path to log}"
}
```

## Error Handling

| Condition | Action |
|-----------|--------|
| Target unreachable | Return status: "error", detail in findings |
| Tool execution fails | Log error, continue with remaining tools |
| No findings | Return status: "complete" with empty findings |

## Operator Log

Write a rolling log to the operator log path. Include:
- Timestamp for each major action
- Tool invocations and their outcomes
- Decision points and rationale
- Discovery markers (**NOTE:**, **DISCOVERY:**, **GOTCHA:**)
```

---

## Directory Structure

```
skills/{skill-slug}/
├── SKILL.md           # The forked skill definition above
├── references/        # Bundled reference docs (optional)
└── tools/             # Utility tools (optional)
```

---

## Forge Runtime

Every forge-generated skill includes these pre-flight steps per `forge-runtime.md`:

1. **Check forge runtime:** `test -d ~/.config/forge` — if missing, stop with "Run: kit use forge && /forge init"
2. **Read context.json:** `~/.config/forge/context.json` for environment awareness (resolver, tools, dataset paths)
3. **Check tool config:** `~/.config/forge/{name}/config.json` — if missing, create from assembler-generated defaults and prompt for first-run confirmation. If present, read silently.
4. **Ensure data directory:** `mkdir -p ~/.local/share/forge/{name}/`
5. **On completion:** Write one JSONL line to `~/.local/share/forge/{name}/ledger.jsonl`

---

## Kit Registration

Register via CLI after committing to forge-armory:

```bash
kit add --name {skill-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path skills/{skill-slug} \
  --type skill \
  --domain security \
  --tags {domain-tag},{task-tag},fork,campaign:{skill-slug} \
  --description "One-line description of what this forked agent does"
```

**Required tag:** `fork` — marks this as a Pattern 1 skill that runs as an autonomous subprocess.

---

## Key Differences from Pattern 0

| Aspect | Pattern 0 | Pattern 1 |
|--------|-----------|-----------|
| Context | Main thread | Forked subprocess |
| `context` field | Not set | `fork` |
| `agent` field | Not set | Persona reference |
| `model` field | Not set | Explicit model selection |
| `user-invocable` | `true` | `false` (invoked by parent) |
| Conversation history | Full access | None — isolated |
| Input | Conversational | Structured $ARGUMENTS |
| Output | Inline in conversation | Structured JSON to file |

---

## Checklist

- [ ] Frontmatter has `context: fork`, `agent:`, `model:`, `allowed-tools:`
- [ ] `user-invocable: false` (forked skills are invoked by parent, not user)
- [ ] $ARGUMENTS block defines structured input contract
- [ ] Workflow has input validation as first phase
- [ ] Output format is structured JSON
- [ ] Error handling table covers failure modes
- [ ] Operator log section specifies logging expectations
- [ ] Kit registration uses --tags with `fork` tag
