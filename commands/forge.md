---
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
description: Security campaign orchestrator. Invokes forge-planner to determine tier and produce a plan, then routes to forge-assembler (Tier 2+) or creates a work order in .sable/work/ (Tier 1).
---

# /forge — Security Campaign Orchestrator

You are a thin orchestration layer. You do NOT plan, analyze, or generate artifacts — you delegate entirely to forge-planner and forge-assembler.

## Step 1: Validate Intent

The user's message is their security intent. If no intent is provided (empty or unclear), ask:

> What security goal do you want to pursue? (e.g., a target to assess, a threat to investigate, a capability to build)

Do NOT proceed until you have a clear intent string.

## Step 2: Invoke Planner

Call the planner skill with the user's security intent:

```
Skill("forge-planner")
```

Pass the user's intent as the conversational argument. Wait for the planner to complete. It will write a plan document to `~/development/projects/forge-armory/plans/`.

## Step 3: Locate the Plan Document

After forge-planner completes, locate the plan it wrote:

```bash
ls -t ~/development/projects/forge-armory/plans/*.md | head -1
```

If the planner surfaced the plan path in its output, use that directly. Otherwise use the command above to find the most recently written plan file. Store this as PLAN_PATH.

## Step 4: Read Tier from Plan

Read PLAN_PATH. Extract the `tier:` field from the YAML frontmatter (integer 1-5).

## Step 5: Route by Tier

### Tier 1 — Direct Execution

The plan contains methodology and commands the practitioner runs directly. Create a work order:

1. Generate a random 4-character alphanumeric ID
2. Write work order to `.sable/work/work-order-forge-<id>.md` with:
   - Plan title and tier
   - Reference to PLAN_PATH
   - Instructions to follow the plan methodology directly
3. Tell the practitioner: "Work order created at `.sable/work/work-order-forge-<id>.md`. This is a Tier 1 plan — follow the methodology commands directly."

<!-- NOTE: /orchestrate-work does not exist yet. Work order is created directly here.
     When /orchestrate-work becomes available, Tier 1 routing should delegate to it instead. -->

### Tier 2-5 — AI-Assisted Composition

Invoke the assembler skill with the plan path:

```
Skill("forge-assembler")
```

Pass PLAN_PATH as the argument. The assembler generates artifacts, commits to the armory, registers with Kit, installs to XDG paths, and returns an assembly report.

Relay the assembly report to the practitioner.

## Step 6: Report

Summarize what happened:

- **Tier 1:** Report the work order path and instruct the practitioner to run the plan methodology directly.
- **Tier 2+:** Report what was assembled, where artifacts were installed, and Kit registration status from the assembler's report.
