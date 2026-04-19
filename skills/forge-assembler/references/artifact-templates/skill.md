# Skill Template

Use this template when generating skill artifacts. Skills are methodology packages loaded into Claude Code sessions. They come in two invocation modes — the mode is a constraint classifier, not a deployment switch.

---

## Invocation Mode Selection

| Mode | When to use | Frontmatter additions |
|------|-------------|----------------------|
| `inline` | Skill requires practitioner interaction (approval gates, clarifying questions, option presentations) | (none beyond base) |
| `forked` | Bounded autonomous work; all inputs known upfront; no mid-run interaction needed | `context: fork`, `agent:`, `model:` |

**Default:** `forked` unless interaction is required by design. A forked skill is invocation-agnostic — it runs correctly in both modes.

**Assembler constraint:** Forked skills must not contain interaction-marker patterns (four categories in `forge-artifacts.md`). The assembler lints for these at Level 1 verification and rejects on hit.

---

## Inline Skill File: SKILL.md

```markdown
---
name: {skill-slug}
description: USE WHEN {trigger conditions}. {What it does in one sentence}.
argument-hint: <target-or-scope>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
user-invocable: true
---

# {Skill Title}

You are a {domain} specialist. {One sentence identity statement}.

## Intent Matrix

| Intent | Workflow |
|--------|----------|
| {intent phrase 1} | -> {workflow-section-1} |
| {intent phrase 2} | -> {workflow-section-2} |

## {Workflow Section 1}

### Phase 1: {Name}

1. {Step with tool reference}
2. {Step with decision point}
3. {Step with output expectation}

### Phase 2: {Name}

1. {Step}
2. {Step}

**Decision point:** {What determines the next action}

## Standards

- {Quality standard 1}
- {Quality standard 2}
- {Output format requirement}

## Output Format

{Description of what this skill produces — structured findings, narrative report, etc.}
```

---

## Forked Skill File: SKILL.md

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
  "findings": [...],
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

## Mode Differences (Reference)

| Aspect | Inline | Forked |
|--------|--------|--------|
| Context | Main thread | Forked subprocess |
| `context` field | Not set | `fork` |
| `agent` field | Not set | Persona reference |
| `model` field | Not set (caller's model) | Explicit model selection |
| `user-invocable` | `true` | `false` (invoked by parent) |
| Conversation history | Full access | None — isolated |
| Input | Conversational / $ARGUMENTS | Structured $ARGUMENTS only |
| Output | Inline in conversation | Structured JSON to file |
| May contain interaction patterns | Yes | No — assembler lint rejects |

---

## Directory Structure

```
skills/{skill-slug}/
├── SKILL.md           # The skill definition above
├── references/        # Bundled reference docs (optional)
│   ├── {ref-1}.md
│   └── {ref-2}.md
├── tools/             # Utility tools (optional)
│   └── {helper}.ts
└── workflows/         # Workflow procedures (optional, inline skills only)
    └── {workflow}.md
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

**Inline skill:**
```bash
kit add --name {skill-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path skills/{skill-slug} \
  --type skill \
  --domain security \
  --tags {domain-tag},{methodology-tag},inline,campaign:{slug} \
  --description "One-line description of what this skill does"
```

**Forked skill:**
```bash
kit add --name {skill-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path skills/{skill-slug} \
  --type skill \
  --domain security \
  --tags {domain-tag},{task-tag},forked,campaign:{slug} \
  --description "One-line description of what this forked agent does"
```

**Tag convention:** `inline` or `forked` replaces the old `loadable` / `fork` tags.

---

## Hook Composition

When the CONOPS specifies scope constraints, rate-limiting, or safety guardrails, produce hook artifacts:

- **PreToolUse hook script** (`tools/{hook-name}.sh`): shell script that checks preconditions. Must be fast and deterministic — no AI calls.
- **Stop hook** (frontmatter entry): prompt-based quality gate, model-evaluated.

Add hook entries to SKILL.md frontmatter using the nested hook syntax:
```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash tools/{hook-name}.sh"
  Stop:
    - hooks:
        - type: prompt
          prompt: "Verify the skill output meets the exit criteria defined in the CONOPS before stopping."
```

---

## Checklist

### Both Modes
- [ ] Frontmatter has `name`, `description` (USE WHEN format), `allowed-tools`
- [ ] Description starts with "USE WHEN"
- [ ] Standards / quality gates section exists
- [ ] Output format is specified
- [ ] Forge runtime pre-flight steps included

### Inline Only
- [ ] `user-invocable: true`
- [ ] Intent matrix maps intents to workflow sections
- [ ] Kit registration uses `inline` tag

### Forked Only
- [ ] Frontmatter has `context: fork`, `agent:`, `model:`, `user-invocable: false`
- [ ] $ARGUMENTS block defines structured input contract
- [ ] Workflow has input validation as first phase
- [ ] Output format is structured JSON
- [ ] Error handling table covers failure modes
- [ ] Operator log section specifies logging expectations
- [ ] **No interaction-marker patterns** (imperative input requests, conditional waits, option presentations, adaptive branching)
- [ ] Kit registration uses `forked` tag
