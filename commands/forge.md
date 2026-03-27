---
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
description: Security campaign orchestrator. Invokes forge-strategist to develop a CONOPS, pressure-tests it, then routes the approved CONOPS to forge-planner (forked) and assembler.
---

# /forge — Security Campaign Orchestrator

Multi-stage orchestration. Coordinates three specialized components: forge-strategist (conversational strategy), hacker pressure test (adversarial validation), and forge-planner (mechanical plan production).

## Stage 1: Validate Intent

The user's message is their security intent. If no intent is provided (empty or unclear), ask:

> What security goal do you want to pursue? (e.g., a target to assess, a threat to investigate, a capability to build)

Do NOT proceed until you have a clear intent string.

## Stage 2: Invoke forge-strategist (Inline)

```
Skill("forge-strategist")
```

The strategist is Pattern 0 — it runs in this conversation. It will:
- Investigate the practitioner's environment (datasets, tools, existing projects)
- Engage the practitioner conversationally to understand intent
- Present a draft CONOPS for approval
- Write the approved CONOPS to `~/.local/share/forge/conops/`

Do NOT add preamble before invoking. Just invoke it. The strategist speaks for itself.

Do NOT treat this as a fork. The strategist runs HERE, in this conversation. You resume after the strategist writes the CONOPS and returns `CONOPS: <path>`.

**Error path:** If the strategist does not return a CONOPS path, do not proceed. Ask the practitioner what they would like to do.

## Stage 3: Pressure Test

After receiving the CONOPS path from the strategist, spawn a forked pressure test to challenge the CONOPS:

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

## Stage 4: CONOPS Approval Gate

Present the combined CONOPS (including pressure test findings) to the practitioner.

STOP. Explicit approval is required before invoking the planner.

**After receiving practitioner response, evaluate:**

- **Clear approval** ("approved", "looks good", "proceed", "go ahead", "yes") → update CONOPS status and proceed to Stage 5.
- **Ambiguous response** ("maybe", "I'm not sure", "let me think", any response that isn't clearly positive) → ask for explicit confirmation: "I need a clear go/no-go before I can run the planner. Approve this CONOPS, or tell me what needs to change."
- **Rejection or revision request** → return to strategist (see error paths below).

Do NOT update CONOPS status or invoke the planner on an ambiguous response.

**Error paths:**
- **Hacker found a fatal flaw** — practitioner decides: return to strategist with hacker findings as additional context, OR revise CONOPS and re-trigger pressure test.
- **Practitioner rejects CONOPS** — return to strategist. The existing CONOPS is context, not discarded. Re-invoke `Skill("forge-strategist")` with the rejection feedback.
- **Practitioner requests partial revision** — re-engage strategist with the specific change request.

**On clear approval**, update the CONOPS frontmatter and verify:
```bash
sed -i '' 's/^status: draft/status: approved/' {conops_path}
grep -q '^status: approved' {conops_path} && echo "CONOPS_APPROVED" || echo "CONOPS_UPDATE_FAILED"
```

If `CONOPS_UPDATE_FAILED`, report the error to the practitioner. Do NOT proceed to Stage 5.

## Stage 5: Invoke forge-planner (Forked)

```
Skill("forge-planner", conops_path)
```

The planner is Pattern 1 forked — it runs in isolation, reads the CONOPS, produces a plan, and returns `PLAN: <path>`.

After the planner returns, read the plan. Extract `tier:` from YAML frontmatter.

**Error path:** If the planner does not return a plan path, do not proceed. Report the failure to the practitioner.

## Stage 6: Route by Tier

Read the plan and extract the tier from frontmatter.

### Tier 1 — Direct Execution

Create a work order in `.sable/work/` with plan title, tier, plan path reference, and instructions to follow the methodology directly.

### Tier 2-5 — AI-Assisted Composition

```
Skill("forge-assembler", plan_path)
```

Pass the plan path. The assembler presents an assembly preview for approval, then generates artifacts, commits to armory, registers with Kit, and installs.

**Error path:** If the practitioner rejects the plan at this stage, the source to fix is the CONOPS — not the plan. Return to the strategist to revise the CONOPS and re-run the forked planner.

## Stage 7: Report

Summarize what was produced:

- **Tier 1:** Work order path and "follow the methodology directly."
- **Tier 2+:** What was assembled, where installed, Kit registration status.
- **CONOPS path:** Always include for reference.
- **Plan path:** Always include for reference.
