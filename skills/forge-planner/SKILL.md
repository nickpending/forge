---
name: forge-planner
description: USE WHEN user invokes /forge or needs a security campaign plan. Engages practitioner to determine tier (1-5), queries Kit for available components, and produces a structured plan document written to forge-armory/plans/.
argument-hint: <security-intent>
allowed-tools: Read, Write, Glob, Grep, Bash
user-invocable: false
---

# forge-planner

ultrathink

Security campaign planner. Takes intent, engages the practitioner conversationally, determines the appropriate tier of AI involvement, queries Kit for available components, manages operator context, and produces a structured plan document.

## When This Skill Fires

| Intent | Keywords | Action |
|--------|----------|--------|
| Plan a campaign | "plan", "campaign", "assess", "investigate", "scan", "recon" | Full planning workflow |
| Quick tool use | "run nmap", "scan this port", "check this CVE" | Tier 1 — produce work order, skip composition |
| Build detection | "detect", "monitor", "alert on", "pipeline for" | Planning workflow with Tier 4-5 bias |
| Revisit plan | "update the plan", "change scope", "add to campaign" | Read existing plan, modify |

**Do not engage** for: general security questions without a target, tool installation help, non-security topics.

## Planning Workflow

### Step 1: Load References

READ the following reference files to ground your planning:

- READ `${CLAUDE_SKILL_DIR}/references/forge-philosophy.md` — five principles governing planner behavior
- READ `${CLAUDE_SKILL_DIR}/references/forge-tiers.md` — tier rubric, signals, disqualifying/promoting signals
- READ `${CLAUDE_SKILL_DIR}/references/forge-patterns.md` — four artifact patterns with selection matrix
- READ `${CLAUDE_SKILL_DIR}/references/plan-schema.md` — plan output format (YAML frontmatter + markdown body)
- READ `${CLAUDE_SKILL_DIR}/references/kit-integration.md` — Kit CLI commands and query timing

All five references must be loaded before proceeding. These files are authoritative — do not paraphrase or reconstruct from memory.

### Step 2: Manage Forge Context

Check if the operator context file exists:

```bash
test -f ~/.config/forge/context.yaml && echo "EXISTS" || echo "ABSENT"
```

**If ABSENT (first engagement):**
1. Create the context directory and initial file:
   ```bash
   mkdir -p ~/.config/forge
   ```
2. READ `${CLAUDE_SKILL_DIR}/references/context-sample.yaml`
3. WRITE the sample schema to `~/.config/forge/context.yaml` as a starting template
4. Tell the practitioner: "Created your forge context at ~/.config/forge/context.yaml with defaults. I'll update it as we go."

**If EXISTS:**
1. READ `~/.config/forge/context.yaml`
2. Use the operator's environment data (tools, infrastructure, skill level, preferences) to inform planning
3. Reference available tools and datasets when building methodology

During conversation, update `~/.config/forge/context.yaml` with any new operator environment data discovered (new tools mentioned, infrastructure details, workflow preferences). Write updates back to the file.

### Step 3: Engage Practitioner

Start a conversation to understand the security intent. Clarify:

- **Target**: What is being assessed? (domain, IP range, application, organization)
- **Scope**: What is in bounds? What is explicitly out of bounds?
- **Constraints**: Time, access level, stealth requirements, rules of engagement
- **Goals**: What does success look like? Report? Detections? Remediation list?

**Practitioner calibration** (Principle 4 from forge-philosophy.md):
- Infer the practitioner's level from conversation style — never ask directly
- Senior: terse exchanges, assumes shared context, mentions specific tools/techniques
- Mid: asks clarifying questions, references frameworks, methodical
- Junior: broader questions, less tool-specific, may need scope help

Calibration affects how you converse (scaffolding density), not what you produce. The plan artifact is identical regardless of practitioner level. The `practitioner_level` field in the plan frontmatter records your inference.

Ask clarifying questions until you have enough information to determine a tier. Do not rush — a well-scoped plan prevents wasted effort downstream.

### Step 4: Determine Tier

Apply the tier determination rubric from `forge-tiers.md`. Evaluate all five signals:

| Signal | Question to answer |
|--------|--------------------|
| Scope | Is this singular/known, or multi-step/adaptive? |
| Judgment | Is AI judgment needed? Where — within guardrails, or strategically throughout? |
| Repeatability | One-off, reusable methodology, or event-triggered? |
| Who drives | Practitioner directing each step, or mission-level with approval gates? |
| Time horizon | Minutes, session, hours/days, or ongoing? |

**Apply disqualifying signals** (cap the tier):
- "I know exactly what I want" — cap at Tier 1
- High-stakes novel target — cap at Tier 2-3
- First time doing this work — cap at Tier 2

**Apply promoting signals** (raise the tier):
- Output of step N changes step N+1 — promote to Tier 4
- "Run every time X happens" — promote to Tier 5
- Requires ordering/prioritization decisions — promote to Tier 2+

State your tier determination and rationale to the practitioner. Example: "This looks like Tier 3 — the recon steps are mechanical (subfinder, httpx, nuclei) but interpreting the results requires judgment. I'd wrap the mechanical parts and have the agent reason over output."

Select the artifact pattern based on the tier (from forge-patterns.md):

| Tier | Primary Pattern |
|------|----------------|
| 1 | Direct execution (no pattern) |
| 2 | Pattern 0 (inline skill) |
| 3 | Pattern 1 (forked skill-agent) |
| 4 | Pattern 2 (agent + skills) |
| 5 | Pattern 3 (orchestrator) |

### Step 5: Query Kit

After tier determination, query Kit for available components. Kit results inform what the assembler can reuse vs. what it must create. They do not influence tier selection.

Run queries based on tier (from kit-integration.md):

```bash
# For Tier 2+: find existing methodology skills
kit list --type skill --domain security 2>/dev/null || echo "KIT_UNAVAILABLE"

# For Tier 3+: find existing wrappers
kit list --type wrapper --domain security 2>/dev/null || echo "KIT_UNAVAILABLE"

# For Tier 4+: find existing agent personas
kit list --type agent --domain security 2>/dev/null || echo "KIT_UNAVAILABLE"

# Search for specific tool wrappers
kit search <tool-name> 2>/dev/null || echo "KIT_UNAVAILABLE"
```

**Graceful degradation**: If any Kit command fails or returns KIT_UNAVAILABLE:
- Set `components_available: []` in the plan frontmatter
- Add a note in the plan body: "Kit was unavailable during planning. The assembler will create all required components."
- Continue producing the plan normally — Kit makes the assembler faster, but is not required

If Kit returns results, populate `components_available` with matching entries. Identify gaps between what is available and what the plan needs — these become `components_needed` entries for the assembler.

### Step 6: Produce Plan

Write the plan document to the forge-armory plans directory:

**Output path:** `~/development/projects/forge-armory/plans/{slug}.md`

**Slug convention:** kebab-case from campaign intent with date suffix. Examples:
- `example-com-recon-2026-03-24.md`
- `internal-network-sweep-2026-03-25.md`
- `cve-2026-1234-detection-2026-03-25.md`

**Before writing**, ensure the output directory exists:
```bash
mkdir -p ~/development/projects/forge-armory/plans
```

**Plan format:** Follow the schema from `plan-schema.md` exactly. The plan has two layers:

1. **YAML frontmatter** — machine-readable assembler contract with all fields from plan-schema.md
2. **Markdown body** — practitioner-facing plan with sections: Objective, Methodology, Infrastructure Requirements, Success Criteria, Results Format

**Frontmatter fields to populate:**
- `forge_version`: "1.0"
- `created`: today's date (ISO format)
- `intent`: practitioner's original request, preserved verbatim
- `tier`: determined in Step 4
- `tier_rationale`: why this tier was selected, referencing rubric signals
- `artifact_pattern`: selected in Step 4 based on tier
- `execution_mode`: one of `interactive`, `automated`, `scheduled` — recommend based on tier
- `practitioner_level`: inferred in Step 3
- `target`: from Step 3 conversation
- `scope`: from Step 3 conversation
- `tools_required`: tools needed for execution
- `components_available`: from Step 5 Kit query (or empty)
- `components_needed`: gaps identified in Step 5
- `validation`: `deferred` for Tier 1-2, `level-1` or `level-2` for Tier 3+
- `status`: `draft`

**Methodology section guidance** (Principle 5 from forge-philosophy.md):
- Embed reasoning in methodology steps — explain why each step matters, not just what to do
- Tag each step as deterministic or judgment-required (Principle 2)
- Adapt scaffolding density to practitioner level (Principle 4) — same rigor, different detail
- Deterministic steps are wrapper candidates; judgment steps stay in skill/agent context

After writing, return the plan path to the invoker:
"Plan written to: ~/development/projects/forge-armory/plans/{slug}.md"

## Standards

### Graceful Degradation

The planner must work under degraded conditions:

| Condition | Behavior |
|-----------|----------|
| Kit unavailable | `components_available: []`, note in plan, continue normally |
| Context file missing | Create from sample template, populate during conversation |
| Context file corrupt | Log warning, create fresh from sample, continue |
| Reference files missing | Log which references failed to load, continue with loaded references |

Never hang, error out, or refuse to produce a plan due to missing optional components.

### Practitioner Calibration

Per Principle 4, calibration affects conversation style only:

| Level | Conversation style |
|-------|--------------------|
| Senior | Terse, high-signal. Skip obvious context. Focus on what is non-obvious about this specific engagement. |
| Mid | Balanced. Explain non-standard decisions. Confirm scope assumptions. |
| Junior | More structured. Explicit decision points. Explain why each step matters. More verification prompts. |

The plan artifact (YAML frontmatter + markdown body) maintains identical analytical rigor regardless of level. A junior following a well-structured plan should produce results close to a senior — that is the leverage.

### Context Management

The operator context at `~/.config/forge/context.yaml` is the practitioner's full environment profile:

- **Tools**: What security tools are installed and available
- **Infrastructure**: Automation platforms (n8n, Tines, SOAR), datasets, custom wordlists, local infrastructure
- **Preferences**: Workflow style, reporting format
- **Skill level**: Inferred by planner, updated over time
- **Campaigns**: Populated by forge-assembler after each campaign execution

Both forge-planner and forge-assembler read and write this file. Never commit it to any git repository (INV-005).

### Plan Output Integrity

- Plans MUST be written to `~/development/projects/forge-armory/plans/` (INV-003). Never write plans to the forge project directory, .sable/, or any other location.
- Plan frontmatter MUST conform to the schema in plan-schema.md. The assembler parses this programmatically.
- Plan status starts as `draft`. The assembler updates to `assembled` after composition. The practitioner updates to `approved` before execution.
