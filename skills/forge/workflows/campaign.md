# Forge Campaign Pipeline

Multi-stage orchestration. Coordinates four specialized components: ferret (disambiguation), forge-strategist (forked autonomous strategy), hacker pressure test (adversarial validation), and forge-planner (mechanical plan production).

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

Capture the practitioner's raw ask verbatim — this exact text is passed to the ferret and strategist.

## Stage 2: Ferret Disambiguation

Spawn the ferret agent with a text-only disambiguation prompt. The ferret analyzes the WORDS only — no file reads, no tool checks, no environment exploration. Pure linguistic analysis of the practitioner's request.

```
Skill("ferret", """
You are analyzing a security practitioner's request BEFORE any investigation begins. Your job is pure linguistic disambiguation — analyze the WORDS, not the environment.

DO NOT read any files. DO NOT run any commands. DO NOT explore any directories. You are analyzing language, not systems.

Here is the practitioner's request:

---
{raw_ask}
---

Produce exactly three sections:

## The Ask
Restate what the practitioner is actually asking for in clear, unambiguous terms. Strip jargon where it obscures intent. If the ask contains multiple goals, separate them.

## Assumptions Present
List every assumption embedded in the request — things taken for granted, unstated prerequisites, implied constraints. For each: state the assumption and why it matters.

## Ambiguities and Gotchas
List everything that is unclear, underspecified, or potentially problematic. For each: state what is ambiguous and what could go wrong if it is resolved incorrectly.

Be thorough. The output of this analysis feeds directly into an autonomous strategist that cannot ask the practitioner questions. Every ambiguity you miss becomes a blind spot.
""")
```

After the ferret returns, capture its full output for use in Stage 3 and Stage 4.

**Error path:** If the ferret does not return output, proceed directly to Stage 4 with an empty ferret output — disambiguation is valuable but not blocking.

## Stage 3: Present Ferret Output to Practitioner

Present the ferret's three sections to the practitioner:

> **Before I send this to the strategist, here's what the ferret found in your request:**
>
> {ferret output — The Ask, Assumptions Present, Ambiguities and Gotchas}
>
> **Anything to add, correct, or clarify? Or should I send this to the strategist as-is?**

The practitioner can:
- Add context or correct assumptions — append their additions to the ferret output before passing to the strategist
- Say "looks right" or "go" — proceed with ferret output as-is

This is optional refinement. If the practitioner just says "go" that is fine — proceed to Stage 4.

## Stage 4: Spawn Forked Strategist

Invoke the strategist as a Pattern 1 forked skill. It runs autonomously — no conversation context from this thread. All input must be provided upfront.

```
Skill("forge-strategist", """
FERRET_OUTPUT:
{ferret_output_with_any_practitioner_additions}

RAW_ASK:
{raw_ask}

CONTEXT_PATH: ~/.config/forge/context.json
""")
```

The strategist runs in isolation: investigates the environment, resolves ambiguities from the ferret output, and writes a CONOPS to `~/.local/share/forge/conops/`.

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

The planner is Pattern 1 forked — it runs in isolation, reads the CONOPS, produces a plan, and returns `PLAN: <path>`.

After the planner returns, read the plan. Extract `tier:` from YAML frontmatter.

**Error path:** If the planner does not return a plan path, do not proceed. Report the failure to the practitioner.

## Stage 9: Route by Tier

Read the plan and extract the tier from frontmatter.

### Tier 1 — Direct Execution

Create a work order in `.sable/work/` with plan title, tier, plan path reference, and instructions to follow the methodology directly.

### Tier 2-5 — AI-Assisted Composition

```
Skill("forge-assembler", plan_path)
```

Pass the plan path. The assembler presents an assembly preview for approval, then generates artifacts, commits to armory, registers with Kit, and installs.

**Error path:** If the practitioner rejects the plan at this stage, the source to fix is the CONOPS — not the plan. Return to the strategist to revise the CONOPS and re-run the forked planner.

## Stage 10: Report

Summarize what was produced:

- **Tier 1:** Work order path and "follow the methodology directly."
- **Tier 2+:** What was assembled, where installed, Kit registration status.
- **CONOPS path:** Always include for reference.
- **Plan path:** Always include for reference.
