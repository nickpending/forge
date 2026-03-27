---
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
description: Security campaign orchestrator. Invokes forge-planner to determine tier and produce a plan, then routes to forge-assembler (Tier 2+) or creates a work order in .sable/work/ (Tier 1).
---

# /forge — Security Campaign Orchestrator

Thin orchestration layer. Validates intent, invokes the planner (inline), then routes the resulting plan to the assembler or work order system.

## Step 1: Validate Intent

The user's message is their security intent. If no intent is provided (empty or unclear), ask:

> What security goal do you want to pursue? (e.g., a target to assess, a threat to investigate, a capability to build)

Do NOT proceed until you have a clear intent string.

## Step 2: Invoke Planner (Inline)

```
Skill("forge-planner")
```

The planner is a Pattern 0 inline skill — it takes over the conversation. It will:
- Investigate the practitioner's environment (datasets, tools, existing projects)
- Engage the practitioner conversationally to understand intent
- Present a draft plan for approval
- Write the approved plan to `~/development/projects/forge-armory/plans/`

Do NOT add preamble before invoking ("Let me kick this to the planner"). Just invoke it. The planner speaks for itself.

Do NOT treat this as a fork. The planner runs HERE, in this conversation. You resume after the planner writes the plan.

## Step 3: Route by Tier

After the planner writes the plan, locate it:

```bash
ls -t ~/development/projects/forge-armory/plans/*.md | head -1
```

Read the plan. Extract `tier:` from YAML frontmatter.

### Tier 1 — Direct Execution

Create a work order in `.sable/work/work-order-forge-<id>.md` (4 random alphanumeric chars) with plan title, tier, plan path reference, and instructions to follow the methodology directly.

<!-- NOTE: /orchestrate-work does not exist yet. Direct work order creation until available. -->

### Tier 2-5 — AI-Assisted Composition

```
Skill("forge-assembler")
```

Pass the plan path. The assembler presents an assembly preview for approval, then generates artifacts, commits to armory, registers with Kit, and installs.

## Step 4: Report

- **Tier 1:** Work order path + "follow the methodology directly."
- **Tier 2+:** What was assembled, where installed, Kit registration status.
