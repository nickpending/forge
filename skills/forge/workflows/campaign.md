# Forge Campaign Pipeline

Multi-stage orchestration. Coordinates four specialized components: sensemaker (intent decomposition with ranked resolutions), forge-strategist (forked autonomous strategy), hacker pressure test (adversarial validation of CONOPS), and forge-planner (mechanical plan production).

## Stage 0: Pre-flight — Forge Runtime

Verify the forge runtime exists before any pipeline stage:

```bash
test -d ~/.config/forge && test -f ~/.config/forge/context.json && echo "FORGE_RUNTIME: OK" || echo "FORGE_RUNTIME: MISSING"
```

If MISSING: run forge init automatically, then continue.

## Stage 1: Validate Intent

The user's message is their security intent. If no intent is provided (empty or unclear), ask:

> What security goal do you want to pursue? (e.g., a target to assess, a threat to investigate, a capability to build)

Do NOT proceed until you have a clear intent string.

Capture the practitioner's raw ask verbatim — this exact text is passed to the sensemaker and strategist.

## Stage 2: Sensemaker Decomposition

Spawn the sensemaker agent (Iris Novak) with the raw ask AND a forge-specific strategy. The sensemaker reads forge context and references, commits to the strongest plausible interpretation, and produces ranked Key Decisions with proposed resolutions — not a list of open questions.

Generate CORRELATION_ID: `sensemaker-forge-{slug-from-raw-ask}-{8 random hex chars}`

```
Agent(subagent_type: "sensemaker", prompt: """
CORRELATION_ID: {correlation_id}
SESSION_ID: {SESSION_ID from hook context}

INTENT:
---
{raw_ask}
---

STRATEGY:

You are decomposing a security practitioner's intent for a forge campaign. Read the following to ground your interpretation:

- `~/.config/forge/context.json` — the practitioner's environment profile (infrastructure, datasets, network constraints, prior campaigns)
- `~/.local/share/forge/conops/` — prior CONOPS documents (if any relate to this intent)
- `~/development/projects/forge/skills/forge-strategist/references/forge-philosophy.md` — the five principles governing when AI adds value
- `~/development/projects/forge/skills/forge-strategist/references/forge-artifacts.md` — six artifact types, four runtimes, complexity rubric, reusability model
- `~/development/projects/forge/skills/forge-strategist/references/forge-timing.md` — timing profiles (T1/T2/T3)
- `~/development/projects/forge/skills/forge-strategist/references/conops-schema.md` — the CONOPS output schema (what the strategist will need from your decomposition)

What to notice while reading:

- **Scope signals** — targets, hosts, datasets, systems, authorization boundaries. What's in scope vs out.
- **Artifact-type signals** — deterministic code (tool/automation_config) vs methodology (skill) vs sustained reasoning (agent) vs orchestration (command/harness). Where does AI actually add value?
- **Runtime signals** — human session (interactive), scheduled Claude Code (unattended), Agent SDK runtime (standalone program), deterministic pipeline (cron/Justfile/n8n). Which process model fits?
- **Consumer signals** — who reads the output? Human operator, downstream automation, another agent?
- **Constraint signals** — timing profile, rate limits, practitioner level, infrastructure limits.
- **Security-specific concerns** — authorization, scope enforcement, sensitive data handling, safety gates that may require hooks.

Resolution orientation:

- Commit to the strongest plausible artifact type, runtime, and scope interpretation — the strategist will build a CONOPS from your decomposition
- Rank Key Decisions by blast radius: artifact type and scope decisions first, output format and integration points second
- Hard cap on Flagged Gaps: maximum 3, only items that genuinely cannot be resolved from context.json + prior CONOPS + the raw ask

Output shape:

Follow the sensemaker's standard 5-section contract (Synthesized Intent, Key Decisions, Flagged Gaps, Artifact Sources, Scope Boundaries). The Synthesized Intent section should be concrete enough that the forge-strategist can build a complete CONOPS from it without needing to ask the practitioner additional questions.

Write report to `{SCOPE_DIR}/agents/reports/sensemaker-forge-{slug}.md`
Write operator log to `{SCOPE_DIR}/agents/operators/sensemaker-forge-{slug}.md`
Return SENSEMAKER_FLAGS per agent format.
""")
```

After the sensemaker returns, parse the SENSEMAKER_FLAGS and READ the report. Store the report path for Stage 4.

**Error path:** If the sensemaker does not return a report, proceed directly to Stage 4 with the raw ask only — decomposition is valuable but not blocking. Note the failure in the campaign summary.

## Stage 3: Present Sensemaker Decomposition to Practitioner

Present the sensemaker's five sections to the practitioner:

> **Before I send this to the strategist, here's how Iris decomposed your intent:**
>
> {sensemaker report — Synthesized Intent, Key Decisions, Flagged Gaps, Artifact Sources, Scope Boundaries}
>
> **The Key Decisions are her calls — approve them, redirect specific ones, or resolve the Flagged Gaps. Then I send it to the strategist.**

The practitioner can:
- **Approve all** — proceed to Stage 4 with sensemaker output as-is
- **Redirect specific Key Decisions** — append overrides to the decomposition; resume the sensemaker if the redirects change the synthesized intent substantially
- **Resolve Flagged Gaps** — append answers to the decomposition before passing to the strategist
- **Say "go"** — proceed as-is

This is the commitment checkpoint. The practitioner's approved decomposition is what feeds the strategist.

## Stage 4: Spawn Forked Strategist

Invoke the strategist as a forked skill. It runs autonomously — no conversation context from this thread. All input must be provided upfront.

```
Skill("forge-strategist", """
SENSEMAKER_REPORT: {sensemaker_report_path}
PRACTITIONER_OVERRIDES:
{any redirects or gap resolutions the practitioner added at Stage 3 — empty if approved as-is}

RAW_ASK:
{raw_ask}

CONTEXT_PATH: ~/.config/forge/context.json
""")
```

The strategist runs in isolation: reads the sensemaker decomposition (the synthesized intent and approved Key Decisions), applies any practitioner overrides, investigates the environment further if needed, and writes a CONOPS to `~/.local/share/forge/conops/`. Because the sensemaker has already committed to tier/runner/scope decisions with the practitioner's approval, the strategist's job is lighter — build the CONOPS from an already-concrete intent rather than disambiguate from scratch.

It returns `CONOPS: <path>`.

**Error path:** If the strategist does not return a CONOPS path, do not proceed. Report the failure to the practitioner and ask what they would like to do.

## Stage 5: Present CONOPS

After receiving the CONOPS path from the strategist:

1. READ the CONOPS file
2. Present the full CONOPS to the practitioner for review

> **The strategist produced this CONOPS. Review it — I'll run a pressure test next, but flag anything that needs revision now.**
>
> {CONOPS content}

The practitioner can:
- Request changes — re-invoke the strategist with the feedback appended to the original arguments
- Accept — proceed to Stage 6

## Stage 6: Pressure Test

Spawn a forked pressure test to challenge the CONOPS:

```
Skill("hacker", """
You are reviewing a Concept of Operations document. Your task is NOT reconnaissance — it is CONOPS critique.

Read the CONOPS at: {conops_path}

Challenge this CONOPS. Return exactly three sections:

## Fragile Assumptions
List every assumption that must be true for this concept to work. For each: state the assumption, and whether it is stated or unstated in the CONOPS.

## Missing Elements
What is the plan missing that a real attacker would exploit, or that would cause the campaign to fail in practice? Be specific.

## Likely Failure Modes
What is most likely to go wrong when this concept is executed? Rank by probability.

Do NOT suggest implementation. Do NOT recommend tools. Focus on concept validity, assumption fragility, and operational gaps.
""")
```

**If the hacker skill is not available** (not installed via Kit), run the pressure test inline instead — read the CONOPS yourself and produce the same three sections (Fragile Assumptions, Missing Elements, Likely Failure Modes) with an adversarial mindset. Focus on concept validity, not implementation.

After the pressure test completes, append findings to the CONOPS file:

```bash
# Read the CONOPS, append pressure test findings
echo "" >> {conops_path}
echo "## Pressure Test Findings" >> {conops_path}
echo "" >> {conops_path}
# (append the three sections from the pressure test output)
```

Write the pressure test output into the `## Pressure Test Findings` section of the CONOPS file.

## Stage 7: CONOPS Approval Gate

Present the combined CONOPS (including pressure test findings) to the practitioner.

STOP. Explicit approval is required before invoking the planner.

**After receiving practitioner response, evaluate:**

- **Clear approval** ("approved", "looks good", "proceed", "go ahead", "yes") → update CONOPS status and proceed to Stage 8.
- **Ambiguous response** ("maybe", "I'm not sure", "let me think", any response that isn't clearly positive) → ask for explicit confirmation: "I need a clear go/no-go before I can run the planner. Approve this CONOPS, or tell me what needs to change."
- **Rejection or revision request** → return to strategist (see error paths below).

Do NOT update CONOPS status or invoke the planner on an ambiguous response.

**Error paths:**
- **Hacker found a fatal flaw** — practitioner decides: return to strategist with hacker findings as additional context, OR revise CONOPS and re-trigger pressure test.
- **Practitioner rejects CONOPS** — return to strategist. The existing CONOPS is context, not discarded. Re-invoke `Skill("forge-strategist")` with the rejection feedback appended to the original arguments.
- **Practitioner requests partial revision** — re-engage strategist with the specific change request.

**On clear approval**, update the CONOPS frontmatter and verify:
```bash
sed -i '' 's/^status: draft/status: approved/' {conops_path}
grep -q '^status: approved' {conops_path} && echo "CONOPS_APPROVED" || echo "CONOPS_UPDATE_FAILED"
```

If `CONOPS_UPDATE_FAILED`, report the error to the practitioner. Do NOT proceed to Stage 8.

## Stage 8: Invoke forge-planner (Forked)

```
Skill("forge-planner", conops_path)
```

The planner is a forked skill — it runs in isolation, reads the CONOPS, produces a plan, and returns `PLAN: <path>`.

After the planner returns, read the plan. Extract `artifact_type:` and `components_needed:` from YAML frontmatter.

**v1 fallback:** If `artifact_type:` is absent but `tier:` is present, treat as a v1 plan. Extract `tier:` and use the old routing (tier 1 = direct, tier 2-5 = assembler). Log deprecation note.

**Error path:** If the planner does not return a plan path, do not proceed. Report the failure to the practitioner.

## Stage 9: Route by Composition Need

Read the plan frontmatter. The routing question is: **does this plan have components the assembler must create?**

### Direct Execution (no assembly needed)

If `components_needed` is empty, or all needed components are already available in Kit (`components_available` covers them):

Create a work order in `.sable/work/` with plan title, artifact type, plan path reference, and instructions to follow the methodology directly.

### Assembly Required

If `components_needed` has items that need creation (not already in `components_available`):

```
Skill("forge-assembler", plan_path)
```

Pass the plan path. The assembler presents an assembly preview for approval, then generates artifacts, commits to armory, registers with Kit, and installs.

**Error path:** If the practitioner rejects the plan at this stage, the source to fix is the CONOPS — not the plan. Return to the strategist to revise the CONOPS and re-run the forked planner.

## Stage 10: Report

Summarize what was produced:

- **Direct execution:** Work order path and "follow the methodology directly."
- **Assembled:** What was assembled, where installed, Kit registration status.
- **CONOPS path:** Always include for reference.
- **Plan path:** Always include for reference.
