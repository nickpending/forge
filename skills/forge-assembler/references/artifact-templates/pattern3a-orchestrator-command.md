# Pattern 3A: Orchestrator Command Template

Use this template when generating prompt-layer orchestrator commands for Tier 5 plans. Pattern 3A orchestrators coordinate multiple Pattern 1 and Pattern 2 agents by chaining Skill() calls within a markdown command. The `/forge` command itself IS a Pattern 3A orchestrator and serves as the reference example.

> Pattern 3B (Agent SDK orchestrator) is backlogged. See flux.

---

## Command File: {command-slug}.md

```markdown
---
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
description: {One-line description of what this orchestrator coordinates end-to-end}
---

# /{command-slug} -- {Orchestrator Title}

You are a thin orchestration layer. You do NOT perform domain work directly -- you delegate to specialized agents via Skill() calls and manage the flow between them.

## Step 1: Validate Input

{Describe what input the orchestrator expects from the user or trigger event.}

If input is missing or unclear, prompt the user:

> {Appropriate prompt for missing input}

Do NOT proceed until input is validated.

## Step 2: Invoke Stage 1 Agent

Call the first agent in the sequence:

\```
Skill("{stage-1-agent-skill}")
\```

Pass {describe what data is passed}. Wait for completion. The agent writes its output to {describe output location or format}.

## Step 3: Approval Gate (if applicable)

{If this stage has an approval gate:}

Present the Stage 1 output summary to the practitioner:

> Stage 1 complete. {Summary of what was produced}. Review and approve to continue, or provide feedback.

Do NOT proceed to Stage 2 until the practitioner approves.

{If no approval gate, remove this step and renumber.}

## Step 4: Invoke Stage 2 Agent

Read the output from Stage 1. Call the next agent:

\```
Skill("{stage-2-agent-skill}")
\```

Pass {describe data handoff -- what Stage 2 receives from Stage 1}. Wait for completion.

## Step N: Additional Stages

{Repeat the pattern: invoke agent, handle data handoff, optional approval gate.}

## Step N+1: Report

Summarize what happened across all stages:

- **Stage 1:** {What was accomplished}
- **Stage 2:** {What was accomplished}
- **Overall:** {End-to-end outcome}
```

---

## Agent Sequencing

The orchestrator defines the order agents execute and what data flows between them:

| Stage | Agent (Skill) | Input | Output | Approval Gate |
|-------|---------------|-------|--------|---------------|
| 1 | `{agent-1-skill}` | {trigger data} | {stage 1 output} | {yes/no} |
| 2 | `{agent-2-skill}` | {stage 1 output} | {stage 2 output} | {yes/no} |
| 3 | `{agent-3-skill}` | {stage 2 output} | {final output} | {yes/no} |

### Data Handoff

Each stage produces output that the next stage consumes. The orchestrator is responsible for:
1. Locating the output from the previous stage (file path, structured data, or conversational output)
2. Passing that output as the argument or context for the next Skill() call
3. Not transforming the data itself -- the orchestrator routes, it does not process

### Approval Gates

| Value | Behavior |
|-------|----------|
| No | Stage output passes directly to next stage invocation |
| Yes | Orchestrator pauses, presents summary to practitioner, waits for approval before continuing |

**When to use approval gates:**
- After triage or prioritization decisions (high-impact choices)
- Before actions that are expensive or irreversible
- At any point where human judgment improves outcomes

**When NOT to use:**
- Between deterministic stages (mechanical operations)
- Where delay defeats the purpose (real-time response)

---

## Kit Registration

Register via CLI after committing to forge-armory:

```bash
kit add --name {command-slug} \
  --repo git@github.com:nickpending/forge-armory.git \
  --path commands/{command-slug} \
  --type command \
  --domain security \
  --tags {domain-tag},{workflow-tag},orchestrator,pattern-3a,campaign:{command-slug} \
  --description "One-line description of what this orchestrator coordinates end-to-end"
```

---

## Component Agents and Skills

Pattern 3A orchestrators do not exist in isolation. Each Skill() reference in the command must point to a real agent or skill. During assembly:

1. **Each component agent** gets its own directory and Kit registration (Pattern 1 or Pattern 2)
2. **Each component skill** gets its own directory and Kit registration (Pattern 0)
3. **The orchestrator command** registers as Kit type `command`
4. All components register individually by their own types

The assembler generates the orchestrator command AND all component agents/skills as separate artifacts in a single assembly batch.

---

## Checklist

- [ ] Command has `allowed-tools` and `description` in frontmatter
- [ ] `allowed-tools` includes `Skill` (required for orchestration)
- [ ] Each stage invokes a specific Skill() with a real skill name
- [ ] All Skill() references point to agents/skills in Kit or the current assembly batch
- [ ] Data handoff between stages is explicit (what output goes where)
- [ ] Approval gates are explicitly declared per stage
- [ ] At least one approval gate exists for orchestrators with high-impact decisions
- [ ] Command follows thin orchestration pattern (delegates, does not do domain work)
- [ ] Kit registration uses --type command
