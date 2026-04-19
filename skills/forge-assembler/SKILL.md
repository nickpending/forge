---
name: forge-assembler
description: USE WHEN forge-planner has produced a plan document and artifacts need to be generated. Takes plan path, generates artifacts per artifact type, commits to forge-armory, registers with Kit, installs to XDG paths.
argument-hint: <plan-path>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill
user-invocable: false
---

# forge-assembler

<!-- IMPORTANT: This skill MUST be invoked inline (NOT forked). Step 3.5 requires practitioner approval before artifact generation. Forked skills cannot interact with the practitioner. -->

ultrathink

Artifact generator for Forge campaigns. Takes a structured plan document produced by forge-planner, generates the correct artifacts for the plan's artifact type, verifies them, commits to the armory, registers with Kit, and installs to XDG paths.

## When This Skill Fires

| Intent | Action |
|--------|--------|
| Assemble a plan | Full 13-step workflow (includes Step 6.5: Level 3 testing) |
| Skill plan | Generate skill (inline or forked per invocation_mode), verify, register |
| Tool plan | Generate deterministic tool, verify, register |
| Agent plan | Generate persona + skill set, verify, register |
| Command plan | Generate orchestrator command + component agents + skills, verify, register |
| Harness plan | Generate Agent SDK project + embedded agents, verify, register |
| Automation config plan | Generate DAG/cron/Justfile, verify (armory-only, no Kit registration) |
| forge-armory missing | Fail fast with clone instructions |
| Verification failure | Report failure, do not commit |

## Pre-flight Checks

Before loading any references or processing the plan, validate the environment.

**Check 1 — forge-armory repo:**

```bash
test -d ~/development/projects/forge-armory && echo "ARMORY: OK" || echo "ARMORY: MISSING"
```

If MISSING: stop immediately. Return this message to the caller:

> forge-armory repo not found. Clone it: `git clone git@github.com:nickpending/forge-armory.git ~/development/projects/forge-armory`

Do not proceed with any assembly steps.

**Check 2 — bun availability:**

```bash
which bun > /dev/null 2>&1 && echo "BUN: OK" || echo "BUN: MISSING"
```

If MISSING: note that Level 2 TypeScript wrapper verification will be skipped. Log the caveat. Continue with all other steps.

## Assembly Workflow

### Step 1: Read Plan Document

Read the file at `<plan-path>` (the argument passed to this skill).

Parse the YAML frontmatter and extract these fields:

| Field | Required | Purpose |
|-------|----------|---------|
| `artifact_type` | YES | Determines which templates to load |
| `components_needed` | YES | What artifacts to generate |
| `validation` | YES | Which verification levels to run |
| `status` | YES | Must be `draft` or `approved` |
| `intent` | YES | Used for commit message and context entry |

Validate all required fields are present. If any are missing, return a failure report listing the missing fields. Do not continue.

If `status` is `assembled`, refuse to reassemble unless the caller explicitly passes an override flag. Return: "Plan already assembled. Pass --force to reassemble."

### Step 2: Load Composition Rules

READ `${CLAUDE_SKILL_DIR}/references/composition-rules.md`

These 15 rules govern all artifact generation decisions in the steps that follow. Apply them during Step 4 (Generate Artifacts). Key rules to keep front of mind:

- **Rule 1**: One skill per methodology concern — split when plan crosses 2+ domains
- **Rule 2**: Wrapper boundary — deterministic = wrapper, reasoning = skill, never mix
- **Rule 4**: Granularity floor — "run this one command" is direct tool use, not a skill
- **Rule 8**: Wrappers are TypeScript only — no bash wrappers
- **Rule 10**: One-off work stays in the work system — do not assemble a skill for single-use tasks

### Step 3: Load Foundations and Artifact Templates

**Load foundations first** — these are the standards you write to:

```
Skill("prompt-foundations")
```
Always load prompt-foundations. Every artifact the assembler generates is a prompt — skills, commands, agent personas are all instructions for a model.

Then load type-specific foundations based on what you're generating:
- Generating skills? → `Skill("skill-foundations")`
- Generating commands? → `Skill("command-foundations")`
- Generating tools? → READ `${CLAUDE_SKILL_DIR}/references/tool-standards.md`
- Generating multiple types? → Load all relevant.

**Load the forge runtime contract:**
- READ `${CLAUDE_SKILL_DIR}/references/forge-runtime.md` — defines where generated artifacts store config, state, and run history. Every artifact you generate must follow this contract.

**Then load artifact templates** from `${CLAUDE_SKILL_DIR}/references/artifact-templates/`:

| Artifact type | Templates to load | Notes |
|---------------|-------------------|-------|
| `tool` | `tool.md` | Deterministic wrapper, TypeScript only |
| `skill` | `skill.md` | Inline or forked per `invocation_mode` in plan |
| `agent` | `agent-persona.md` + `agent-skills.md` | Persona + 2-4 inline skills |
| `command` | `command.md` | Orchestrator command + component agents/skills |
| `automation_config` | `automation-config.md` | DAG/cron/Justfile, armory-only |
| `harness` | `harness.md` | Agent SDK project + embedded agents |

READ each template file. Templates are authoritative -- do not deviate from their structure.

**Direct execution early return:** If the plan's `components_needed` is empty and all required components are Kit-available, skip artifact generation. Write a work order to `${WORK_DIR}/` following the standard work order format, then return the assembly report. Do not proceed to Step 4.

### Step 3.5: Present Assembly Plan

Before generating any artifacts, present what you will build to the practitioner:

**Assembly preview:**
- Artifact type: {artifact_type}, Runtime: {runtime} from the plan
- Components to generate:
  - {name}: {type} — {purpose} — reusability: {reusability} → writes to forge-armory/{path}
  - (repeat for each component)
- Verification: Level {validation} per verification-checklist.md
- Kit registration: {N} kit-eligible artifacts via `kit add` (with bundle tags where parent-scoped)
- Kit install: {N} artifacts via `kit use`
- Armory-only: {N} components (automation configs + private components)

STOP and wait for practitioner approval before proceeding. Do NOT generate artifacts without explicit confirmation.

**Approval signals:** "yes", "go ahead", "proceed", "do it" → proceed to Step 4.
**Redirect signals:** Practitioner wants changes → adjust component list, re-present.

### Step 4: Generate Artifacts

Apply the loaded templates, composition rules, and foundation standards to generate artifacts. Follow prompt engineering principles from prompt-foundations. Follow structural patterns from skill-foundations (for skills) and command-foundations (for commands).

**Forge runtime contract (from forge-runtime.md):** Every generated artifact must include the forge runtime preamble, use forge XDG paths for config and state, and write a ledger entry on completion. When generating an artifact:
1. Include the bootstrap preamble (check forge runtime, read context.json, check/create tool config)
2. Generate a default `config.json` at `~/.config/forge/{name}/` with sensible defaults from the plan's domain context. Set `confirmed: false`.
3. Include a ledger write at the end of each run (one JSONL line to `~/.local/share/forge/{name}/ledger.jsonl`)
4. Shared environment data (resolver, dataset paths, tool inventory) comes from context.json — never duplicate into per-tool config

Artifact-type-specific instructions:

**Tool:**
- Generate using `tool.md` template
- Assess complexity: default to single-file Bun (< ~500 lines, no external consumers, no npm deps). Escalate to multi-file only when complexity signals warrant it.
- Directory: `tools/{tool-name}/`
- Tool handles the deterministic operation (same input, same output)

**Skill (inline mode):**
- Generate using `skill.md` template (inline section)
- One skill per methodology concern (composition rule 1)
- Apply naming conventions from composition-rules.md (methodology-focused slugs)
- Each skill gets its own directory: `skills/{skill-slug}/`
- Include `references/` subdirectory if the skill bundles reference documents
- Set `user-invocable: true` in frontmatter

**Skill (forked mode):**
- Generate using `skill.md` template (forked section)
- Directory: `skills/{skill-slug}/`
- Set `context: fork`, `user-invocable: false` in frontmatter
- **Assembler lint:** verify no interaction-marker patterns present (four categories in `forge-artifacts.md`). Reject on hit — do not commit.

**Agent:**
- Generate persona using `agent-persona.md` template:
  - Directory: `agents/{role-slug}/`
  - Name must be role-based, not campaign-specific (composition rule 7)
  - Frontmatter: `name`, `model`, `allowed-tools`
  - Content: Identity, Expertise, Behavioral Constraints, Communication Style
- Generate 2-4 inline skills using `agent-skills.md` guidance:
  - Each skill covers a distinct methodology concern
  - Each gets its own directory under `skills/`
  - All tagged `inline` (not `forked`)
  - Verify no overlap between skills (each covers separate domain)

**Command:**
- Generate orchestrator command using `command.md` template:
  - Command chains Skill() calls to coordinate component agents
  - Generate component agents and their skills as separate artifacts
  - Each component artifact gets its own directory and is registered individually with Kit
  - Apply reusability rubric per inner component (see plan's `components_needed[].reusability`)
  - Verify all Skill() references point to real agents/skills in Kit or the current assembly batch

**Automation Config:**
- Generate using `automation-config.md` template
- Co-locate with parent tool in `tools/{tool-name}/`
- **Armory-only** — do NOT `kit add` this artifact
- Verify control flow is deterministic (no LLM-driven routing)

**Harness:**
- Generate Agent SDK project using `harness.md` template
- Use committed defaults: TypeScript + bun primary (plan can override to Python + uv)
- Populate from plan-schema fields: `harness_agents`, `harness_schedule`, `harness_env`
- Directory: `harnesses/{harness-name}/`
- Embed agent personas in `.claude/agents/` from `harness_agents` list
- Apply reusability rubric to inner agents (reusable → Kit-register independently; parent-scoped → Kit-register with `bundle:harness:{harness-name}` tag; private → embed, no Kit)
- Verification: `bun run tsc --noEmit` (TS) or `python -m py_compile` + import smoke (Python)

### Step 5: Level 1 Verification

READ `${CLAUDE_SKILL_DIR}/references/verification-checklist.md` for full verification criteria.

For each generated **skill** artifact:

```
Skill("artifact-foundations:skill-foundations")
```

Pass the artifact directory path. If the invocation returns errors or no result, treat as FAIL.

For each generated **command** artifact:

```
Skill("artifact-foundations:command-foundations")
```

For **agent personas**: manually verify frontmatter has `name`, `model`, `allowed-tools` and content has Identity, Expertise, Behavioral Constraints, Communication Style sections.

**On Level 1 FAIL:** Report the specific failures (missing fields, malformed frontmatter, broken references). Do NOT proceed to Step 6. Do NOT commit. Return failure report with:
- Which artifact failed
- Which specific check failed
- The error message or missing element
- Suggested fix

### Step 6: Level 2 Verification

For **tools (TypeScript):**

Detect tool structure — single-file or multi-file:

```bash
cd <tool-dir>
if [ -f cli.ts ]; then
  # Multi-file: build from cli.ts entry point
  bun build ./cli.ts --outdir /tmp/forge-verify-${RANDOM} --target bun 2>/dev/null && echo "L2: PASS" || echo "L2: FAIL"
else
  # Single-file: find the .ts file and build it
  TS_FILE=$(ls *.ts 2>/dev/null | head -1)
  bun build ./${TS_FILE} --outdir /tmp/forge-verify-${RANDOM} --target bun 2>/dev/null && echo "L2: PASS" || echo "L2: FAIL"
fi
rm -rf /tmp/forge-verify-*/
```

If bun is not available (flagged in pre-flight), skip with notation: "Level 2 wrapper verification skipped — bun not in PATH."

For **skills:** verify each file referenced in SKILL.md instructions exists within the artifact directory. For each reference path found in the skill content:

```bash
test -f <skill-dir>/<referenced-path> && echo "REF: OK" || echo "REF: MISSING — <path>"
```

Report any broken references.

For **agent personas:** verify against the format spec in composition-rules.md — `name`, `model`, `allowed-tools` in frontmatter; Identity, Expertise, Behavioral Constraints, Communication Style sections in content.

For **command orchestrators:** verify all Skill() references in the orchestrator point to real agents/skills in Kit or the current assembly batch.

**On Level 2 FAIL:** Report specific failures with fix guidance. Do NOT proceed to Step 7. Do NOT commit.

### Step 6.5: Level 3 Execution Testing

Per the test-before-ship requirement in forge-runtime.md, execute each verified artifact before committing:

**Tools (TypeScript):**
```bash
# Verify help/usage output exists
bun {tool-name}.ts --help 2>&1 | grep -q "Usage:" && echo "L3: HELP OK" || echo "L3: HELP FAIL"
# Derive test input from the tool's --help and the plan intent. Use minimal valid input
# that exercises the tool's primary code path (e.g., a single target, a small input file).
OUTPUT=$(bun {tool-name}.ts {derived-test-input} 2>/dev/null)
echo $OUTPUT | bun -e "JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'))" && echo "L3: JSON OK" || echo "L3: JSON FAIL"
```

**Justfile targets:**
```bash
just --dry-run {target} && echo "L3: JUSTFILE OK" || echo "L3: JUSTFILE FAIL"
```

**n8n workflow JSON:**
```bash
# Validate against n8n workflow schema
bun -e "JSON.parse(require('fs').readFileSync('{workflow.json}','utf8')); console.log('L3: JSON VALID')" || echo "L3: JSON INVALID"
```

**Skills, commands, agents:** Level 3 not applicable — structural validation in Level 1+2 is sufficient.

**On Level 3 FAIL:** Report which artifact failed and the error. Do NOT proceed to Step 7. Do NOT commit.

### Step 7: Write Artifacts to forge-armory

Write verified artifacts to the correct subdirectories in `~/development/projects/forge-armory/`:

| Artifact type | Armory path |
|---------------|-------------|
| Skills | `skills/{skill-name}/` |
| Tools | `tools/{tool-name}/` |
| Agents | `agents/{role-slug}/` |
| Commands | `commands/{command-slug}/` |
| Harnesses | `harnesses/{harness-name}/` |

Commands orchestrate via Skill() chain — register via Kit as type `command`. Harnesses are Agent SDK projects — register via Kit as type `harness`. Component agents and skills register individually by their own types. Automation configs co-locate with their parent tool (armory-only).

**Automation config co-location:** For each tool that has associated automation configs (Justfile, cron config, n8n workflow JSON), write the configs alongside the tool in `tools/{tool-name}/`. Do NOT create a separate `automations/` directory. See composition-rules.md Rule 13.

**Plans directory:** `plans/` is written by forge-planner (Step 5), not the assembler. It exists in the armory but is not part of this step's scope.

Ensure output directories exist before writing:

```bash
mkdir -p ~/development/projects/forge-armory/<target-path>
```

### Step 8: Git Commit to forge-armory

```bash
cd ~/development/projects/forge-armory && \
git add . && \
git commit -m "feat(campaign): <plan-intent-slug>"
```

One commit per campaign. All artifacts for this plan go in a single commit. Derive the commit message from the plan's `intent` field — extract key words, kebab-case them.

Example: intent "do recon on example.com" becomes `feat(campaign): example-com-recon`.

Never split a campaign across multiple commits. Never commit unverified artifacts (Steps 5-6 must pass first).

After successful commit, update the plan's status from `draft` to `assembled`:
```bash
sed -i '' 's/^status: draft/status: assembled/' <plan-path>
```

### Step 9: Kit Registration

Run `kit add` for each artifact. Kit maintains a centralized YAML catalog at a git-hosted repo — there are no sidecar manifest files. Registration happens via CLI flags.

**Type mapping:**

| Forge artifact type | `kit add --type` flag |
|--------------------|-----------------------|
| `skill` | `--type skill` |
| `tool` | `--type tool` |
| `agent` | `--type agent` |
| `command` | `--type command` |
| `harness` | `--type harness` |

**Kit eligibility:** Only register artifacts that are Kit-eligible types (skill, tool, agent, command, harness). Automation configs (Justfiles, cron files, n8n workflows) are committed to armory in Step 7 alongside their tool but are NOT passed to `kit add`. Private-reusability components are co-located in their parent's directory and not Kit-registered. See composition-rules.md Rule 13 and Rule 14.

**Campaign tagging:** All artifacts from the same campaign share a campaign tag derived from the plan intent slug. This enables `kit list --tags campaign:{slug}` to return all artifacts from a campaign as a group.

**Tagging by invocation mode and reusability:**

| Artifact | Required tags |
|----------|--------------|
| Inline skills | `inline`, `campaign:{slug}` |
| Forked skills | `forked`, `campaign:{slug}` |
| Tools | `campaign:{slug}` |
| Agent personas | `campaign:{slug}` |
| Commands | `orchestrator`, `campaign:{slug}` |
| Harnesses | `orchestrator`, `campaign:{slug}` |
| Parent-scoped components | add `bundle:{parent-type}:{parent-slug}` to the above |

**Full command per artifact:**

```bash
kit add --name <name> \
  --repo git@github.com:nickpending/forge-armory.git \
  --path <subdir>/<name> \
  --type <type> \
  --domain security \
  --tags <pattern-tags>,campaign:<intent-slug> \
  --description "<one-line description>"
```

**On partial failure:** If `kit add` fails for any artifact, log which artifacts succeeded before the failure. The user can re-run `kit add` for the failed artifacts. Continue assembling — do not abort the entire campaign for a single registration failure.

### Step 10: Kit Install

Install each artifact to its XDG location:

```bash
kit use <name>
```

Install locations by type:

| Kit type | Install location |
|----------|-----------------|
| `skill` | `~/.claude/skills/<name>/` |
| `tool` (wrapper) | `~/.local/bin/` |
| `agent` | `~/.config/sable/agents/` |
| `command` | `~/.claude/commands/<name>.md` |
| `harness` | User workspace dir (e.g. `~/forge-harnesses/<name>/`); `kit use` clones + installs deps |

If `kit use` fails for any artifact, log the failure with the artifact name. The user can re-run manually.

### Step 11: Update context.json

Check if the context file exists:

```bash
test -f ~/.config/forge/context.json && echo "EXISTS" || echo "ABSENT"
```

**If ABSENT:** Create it:

```bash
mkdir -p ~/.config/forge
```

Write the initial schema:

```json
{"operator": {}, "campaigns": []}
```

Then append the campaign entry.

**If EXISTS:** READ the entire file. Parse the JSON. Append the campaign entry to the `campaigns` array. WRITE the entire file back.

**Do not use shell append operators** (`echo >>` or `cat >>`). They corrupt YAML indentation. Always READ the full file, insert the campaign entry into the campaigns list, and WRITE the whole file back.

Campaign entry format:

```yaml
  - name: "<plan-intent-slug>"
    date: "<ISO date today>"
    artifact_type: <artifact_type>
    runtime: <runtime>
    artifacts:
      - name: <artifact-name>
        type: <artifact-type>
        location: "forge-armory/<subdir>/<name>/"
    status: complete
```

### Step 12: Return Assembly Report

Output a structured summary to the caller covering:

- **Plan processed:** path, intent, artifact type, runtime
- **Artifacts generated:** name, type, armory path for each
- **Verification results:** Level 1 and Level 2 pass/fail per artifact
- **Git commit:** SHA and message
- **Kit registration:** which artifacts registered, which (if any) failed
- **Kit install:** which artifacts installed, which (if any) failed
- **Context update:** confirmed or skipped
- **Warnings:** any skipped checks (bun missing, Kit degradation)

## Standards

### Graceful Degradation

The assembler must work under degraded conditions:

| Condition | Behavior |
|-----------|----------|
| forge-armory not cloned | Fail immediately, print clone instructions |
| bun not in PATH | Skip Level 2 wrapper verification, note in report |
| Kit unavailable | Skip `kit add` + `kit use`, note in report, artifacts still committed to armory |
| context.json corrupt | Log warning, create fresh file with initial schema, continue |

Never hang, error out silently, or leave partial state without reporting it.

### Commit Discipline

- **Never commit unverified artifacts.** Level 1 and Level 2 verification block commit. This is the primary quality gate.
- **One commit per campaign.** Never split a campaign across multiple commits.
- **Direct execution exception.** Plans with no components to build skip the armory entirely — write to the work order system and return.

### Artifact Integrity

- Every skill must pass `artifact-foundations:skill-foundations` before commit
- Every command must pass `artifact-foundations:command-foundations` before commit
- Every wrapper must compile with `bun build` before commit (when bun is available)
- Every command or harness orchestrator must reference only agents/skills that exist in Kit or are included in the current assembly batch

### Context Management

Both forge-strategist and forge-assembler read and write `~/.config/forge/context.json`. The forge-planner reads the CONOPS only — it does not touch context.json. Never commit this file to any git repository. It contains operator-specific environment data.
