# Forge Artifact Patterns

Forge produces different artifact types depending on the work. Four patterns, each with different execution characteristics. The planner selects the pattern based on tier and work shape.

---

## Pattern 0: Inline Skill

The most basic pattern. Instructions loaded into the current conversation. The main thread follows them. No fork, no subprocess, no persona.

```yaml
---
name: recon-methodology
description: Structured recon methodology for domain targets
---
# Steps to follow...
```

**Characteristics:**
- Runs in main thread, shares conversation context
- No isolation — sees everything, can reference prior discussion
- The planner itself (`/forge`) is Pattern 0
- Loadable INTO a Pattern 2 agent as knowledge/methodology

**When to use:** Teaching the model how to do something in the current session. Methodology skills, taxonomies, decision frameworks.

---

## Pattern 1: Forked Skill-as-Agent

A skill that runs as an autonomous subprocess with its own model, tools, and scoped context. The skill IS the agent.

```yaml
---
name: http-sweep
description: Sweep IP list for HTTP services, return structured results
context: fork
agent: <persona-name>
model: sonnet
allowed-tools: Read, Bash, Grep
---
# Self-contained instructions...
```

**Characteristics:**
- Runs in isolation — no conversation context from parent
- Own model selection (haiku for cheap grunt work, sonnet for analysis)
- Scoped tool access
- Returns structured output to parent
- Writes own operator logs
- Does not bloat parent context

**When to use:** Bounded, well-defined tasks. Sweeps, scans, searches, validation checks. The task is clear enough that the agent does not need conversation history.

---

## Pattern 2: First-Class Agent with Loadable Skills

A persistent agent with its own persona/identity that invokes Pattern 0 skills as capabilities. The agent is the reasoner, the skills are the knowledge it draws on.

**Agent definition** (persona + identity):
```
Name: <persona-name>
Archetype: security researcher / analyst / etc
Personality traits: ...
Communication style: ...
```

**Loads Pattern 0 skills:**
- `recon-methodology` — how to approach recon
- `classification-taxonomy` — how to classify findings
- `triage-output-schema` — how to structure output

**Characteristics:**
- Agent has a persona (WHO) separate from instructions (WHAT TO DO)
- Skills provide knowledge/methodology — agent provides reasoning
- Sustained reasoning over complex problems
- Can invoke multiple skills during execution

**When to use:** Complex work requiring sustained judgment. Investigation, analysis, multi-step reasoning where the agent needs to adapt strategy.

---

## Pattern 3: Multi-Agent Orchestrator

Coordinates multiple Pattern 1 and Pattern 2 agents. Manages handoffs, data contracts, and approval gates. Has two execution modes:

**Pattern 3A -- Prompt-layer orchestration:**
- A command (markdown) that chains Skill() calls to coordinate agents
- Runs in Claude Code, same as `/forge` itself
- Registers in Kit as type `command`
- Example: the `/forge` command, Sable's orchestrate-work

**Pattern 3B -- Code-layer orchestration:**
- An Agent SDK application (TypeScript) that spawns agents programmatically
- Runs standalone, on cron, or event-triggered
- Registers in Kit as type `tool`
- Example: Sigil's N-Day Response system
- Built via `agent-sdk-dev:new-sdk-app` skill or Forge work orders
- **Backlogged** -- added to flux for future implementation

**Characteristics (both modes):**
- Defines agent sequence, handoff data contracts, approval gates
- Each coordinated agent can be Pattern 1 or Pattern 2
- The orchestrator itself is an agent whose job is coordination, not direct domain work

**When to use:** Repeatable multi-agent workflows. Event-triggered responses. Work that requires different specialized agents at different stages.

---

## Pattern Selection by Tier

| Tier | Primary Pattern | Why |
|------|----------------|-----|
| 1 | Direct execution (no pattern) | Just run the tool |
| 2 | Pattern 0 (inline skill) | Teach methodology, human drives |
| 3 | Pattern 1 (forked skill-agent) | Bounded deterministic + interpretation |
| 4 | Pattern 2 (agent + skills) | Sustained investigation, adaptive reasoning |
| 5 | Pattern 3 (orchestrator) | Multi-agent coordination (3A: command, 3B: Agent SDK app) |

---

## Kit Tracking by Pattern

Every artifact in the Forge catalog tracks its pattern:

- **Pattern 0 skills:** tagged `loadable` — can be invoked inline or loaded into Pattern 2 agents
- **Pattern 1 skills:** tagged `fork` — self-contained agent subprocess
- **Pattern 2 agents:** separate definitions referencing Pattern 0 skills
- **Pattern 3A orchestrators:** commands coordinating Pattern 1/2 agents (Kit type: `command`)
- **Pattern 3B orchestrators:** Agent SDK apps coordinating agents (Kit type: `tool`, backlogged)

The assembler uses these tags to make composition decisions.
