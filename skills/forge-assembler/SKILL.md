---
name: forge-assembler
description: USE WHEN forge-planner has produced a plan document and artifacts need to be generated. Takes plan path, generates tier-appropriate artifacts, commits to forge-armory, registers with Kit, installs to XDG paths.
argument-hint: <plan-path>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill
user-invocable: false
---

# forge-assembler

<!-- IMPORTANT: This skill MUST be invoked inline (NOT forked). Step 3.5 requires practitioner approval before artifact generation. Forked skills cannot interact with the practitioner. -->

ultrathink

Artifact generator for Forge campaigns. Takes a structured plan document produced by forge-planner, generates the correct artifacts for the plan's tier and pattern, verifies them, commits to the armory, registers with Kit, and installs to XDG paths.

## When This Skill Fires

| Intent | Action |
|--------|--------|
| Assemble a plan | Full 12-step workflow |
| Tier 2 plan (Pattern 0) | Generate skill, verify, register |
| Tier 3 plan (Pattern 1) | Generate forked skill + wrapper, verify, register |
| Tier 4 plan (Pattern 2) | Generate persona + skill set, verify, register |
| Tier 5 plan (Pattern 3A) | Generate orchestrator command + component agents + skills, verify, register |
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
| `tier` | YES | Determines which templates to load |
| `artifact_pattern` | YES | Confirms pattern selection |
| `components_needed` | YES | What artifacts to generate |
| `validation` | YES | Which verification levels to run |
| `status` | YES | Must be `draft` or `approved` |
| `intent` | YES | Used for commit message and context entry |

Validate all required fields are present. If any are missing, return a failure report listing the missing fields. Do not continue.

If `status` is `assembled`, refuse to reassemble unless the caller explicitly passes an override flag. Return: "Plan already assembled. Pass --force to reassemble."

### Step 2: Load Composition Rules

READ `${CLAUDE_SKILL_DIR}/references/composition-rules.md`

These 12 rules govern all artifact generation decisions in the steps that follow. Apply them during Step 4 (Generate Artifacts). Key rules to keep front of mind:

- **Rule 1**: One skill per methodology concern — split when plan crosses 2+ domains
- **Rule 2**: Wrapper boundary — deterministic = wrapper, reasoning = skill, never mix
- **Rule 4**: Granularity floor — "run this one command" is Tier 1, not a skill
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
- Generating both? → Load both.

**Load the forge runtime contract:**
- READ `${CLAUDE_SKILL_DIR}/references/forge-runtime.md` — defines where generated artifacts store config, state, and run history. Every artifact you generate must follow this contract.

**Then load artifact templates** from `${CLAUDE_SKILL_DIR}/references/artifact-templates/`:

| Tier | Pattern | Templates to load | Notes |
|------|---------|-------------------|-------|
| 1 | -- | None | Produce work order only. Return early after Step 4. |
| 2 | 0 | `pattern0-skill.md` | One skill per methodology concern |
| 3 | 1 | `pattern1-forked-skill.md` + `pattern1-ts-wrapper.md` | Skill + wrapper are separate artifacts |
| 4 | 2 | `pattern2-agent-persona.md` + `pattern2-agent-skills.md` | Persona + 2-4 Pattern 0 skills |
| 5 | 3A | `pattern3a-orchestrator-command.md` | Orchestrator command + component agents/skills |

READ each template file. Templates are authoritative -- do not deviate from their structure.

**Tier 1 early return:** Tier 1 plans skip artifact generation, armory commits, Kit registration, and context update entirely. Write a work order to `${WORK_DIR}/` following the standard work order format, then return the assembly report. Do not proceed to Step 4.

### Step 3.5: Present Assembly Plan

Before generating any artifacts, present what you will build to the practitioner:

**Assembly preview:**
- Tier {tier}, Pattern {pattern} from the plan
- Components to generate:
  - {name}: {type} — {purpose} → writes to forge-armory/{path}
  - (repeat for each component)
- Verification: Level {validation} per verification-checklist.md
- Kit registration: {N} artifacts via `kit add`
- Kit install: {N} artifacts via `kit use`

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

Tier-specific instructions:

**Tier 2 (Pattern 0 — Inline Skill):**
- Generate one skill per methodology concern (composition rule 1)
- Use the `pattern0-skill.md` template for structure
- Apply naming conventions from composition-rules.md (methodology-focused slugs)
- Each skill gets its own directory: `skills/{skill-slug}/`
- Include `references/` subdirectory if the skill bundles reference documents
- Set `user-invocable: true` in frontmatter

**Tier 3 (Pattern 1 — Forked Skill + Tool):**
- Generate the tool using `pattern1-ts-wrapper.md` template:
  - Assess complexity: default to single-file Bun (< ~500 lines, no external consumers, no npm deps). Escalate to multi-file only when complexity signals warrant it.
  - Directory: `tools/{tool-name}/`
  - Tool handles the deterministic operation (same input, same output)
- Generate the forked skill using `pattern1-forked-skill.md` template:
  - Directory: `skills/{skill-name}/`
  - Skill handles reasoning and judgment over tool output
  - Set `context: fork`, `user-invocable: false` in frontmatter
- These are separate artifacts registered individually with Kit

**Tier 4 (Pattern 2 — Agent + Skills):**
- Generate agent persona using `pattern2-agent-persona.md` template:
  - Directory: `agents/{role-slug}/`
  - Name must be role-based, not campaign-specific (composition rule 7)
  - Frontmatter: `name`, `model`, `allowed-tools`
  - Content: Identity, Expertise, Behavioral Constraints, Communication Style
- Generate 2-4 Pattern 0 skills using `pattern2-agent-skills.md` guidance:
  - Each skill covers a distinct methodology concern
  - Each gets its own directory under `skills/`
  - All tagged `loadable` (not `fork`)
  - Verify no overlap between skills (each covers separate domain)

**Tier 5 (Pattern 3A — Orchestrator Command):**
- Generate orchestrator command using `pattern3a-orchestrator-command.md` template:
  - Command chains Skill() calls to coordinate component agents
  - Generate component agents (Pattern 1/2) and their skills as separate artifacts
  - Each component artifact gets its own directory and is registered individually with Kit
  - Verify all Skill() references point to real agents/skills in Kit or the current assembly batch

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

For **Pattern 3A orchestrator commands:** verify all Skill() references in the orchestrator point to real agents/skills in Kit or the current assembly batch.

**On Level 2 FAIL:** Report specific failures with fix guidance. Do NOT proceed to Step 7. Do NOT commit.

### Step 7: Write Artifacts to forge-armory

Write verified artifacts to the correct subdirectories in `~/development/projects/forge-armory/`:

| Artifact type | Armory path |
|---------------|-------------|
| Skills | `skills/{skill-name}/` |
| Tools | `tools/{tool-name}/` |
| Agents | `agents/{role-slug}/` |
| Commands | `commands/{command-slug}/` |

Pattern 3A orchestrators are commands -- register via Kit as type `command`. Component agents and skills register individually by their own types.

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
| `wrapper` (tool) | `--type tool` |
| `agent` | `--type agent` |
| `command` | `--type command` |

**Campaign tagging:** All artifacts from the same campaign share a campaign tag derived from the plan intent slug. This enables `kit list --tags campaign:{slug}` to return all artifacts from a campaign as a group.

**Pattern-specific tags:**

| Pattern | Required tags |
|---------|--------------|
| Pattern 0 skills | `loadable`, `campaign:{slug}` |
| Pattern 1 forked skills | `fork`, `campaign:{slug}` |
| Pattern 1 tools | `campaign:{slug}` |
| Pattern 2 agent personas | `campaign:{slug}` |
| Pattern 3A orchestrators | `campaign:{slug}` |

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
    tier: <tier>
    artifacts:
      - name: <artifact-name>
        type: <artifact-type>
        location: "forge-armory/<subdir>/<name>/"
    status: complete
```

### Step 12: Return Assembly Report

Output a structured summary to the caller covering:

- **Plan processed:** path, intent, tier, pattern
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
- **Tier 1 exception.** Tier 1 plans skip the armory entirely — write to the work order system and return.

### Artifact Integrity

- Every skill must pass `artifact-foundations:skill-foundations` before commit
- Every command must pass `artifact-foundations:command-foundations` before commit
- Every wrapper must compile with `bun build` before commit (when bun is available)
- Every Pattern 3A orchestrator command must reference only agents/skills that exist in Kit or are included in the current assembly batch

### Context Management

Both forge-strategist and forge-assembler read and write `~/.config/forge/context.json`. The forge-planner reads the CONOPS only — it does not touch context.json. Never commit this file to any git repository. It contains operator-specific environment data.
