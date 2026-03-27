# Forge Tiers: Five Levels of AI Involvement

Every security task has a tier determined by scope, judgment required, repeatability, and who drives. Tiers are not fixed to task types — "recon" could be Tier 1 (quick scan) or Tier 4 (full investigation) depending on context. Tier selection is part of the planner's expertise.

---

## Tier 1: Direct Tool Use

The model uses a tool directly. No skill, no wrapper. Pair-programming in a terminal.

| Aspect | Detail |
|--------|--------|
| Requirements | Tool installed, model has access |
| Infrastructure | Permissions only |
| AI role | Operator — runs tool, interprets output |
| Human role | Directing — "scan this" |
| Output artifact | Work order or direct execution |
| Example | "Run nmap against 10.0.0.1" |

**When to use:** You know exactly what you want, the task is singular, there is nothing to compose. Also valuable for maintaining practitioner expertise (Principle 5).

---

## Tier 2: Guided Tool Use

The model knows the tools but not your methodology. A skill teaches it your approach.

| Aspect | Detail |
|--------|--------|
| Requirements | A skill that encodes methodology |
| Infrastructure | Skill in catalog, loaded into agent context |
| AI role | Practitioner — follows methodology, applies judgment within guardrails |
| Human role | Intent + review |
| Output artifact | Skill definition (select or generate) |
| Example | "Do recon on this domain" with a recon methodology skill loaded |

**When to use:** The task has a known approach but requires judgment in execution — ordering, prioritization, interpretation of results.

---

## Tier 3: Wrapped Execution

Deterministic tools handle mechanical operations, return structured output. AI consumes and interprets, does not orchestrate mechanics.

| Aspect | Detail |
|--------|--------|
| Requirements | Wrapper tools with structured JSON I/O |
| Infrastructure | Tools installed, accessible to agent, deterministic behavior |
| AI role | Consumer — interprets results, decides next action |
| Human role | Intent |
| Output artifact | Wrapper tool spec + skill |
| Example | Tool searches Nuclei templates for a CVE, returns JSON. Agent interprets findings. |

**When to use:** The mechanical part should never vary — same inputs always produce same outputs. AI's value is in what comes after the output.

### Tool Requirements (for the plan)

When the plan specifies a tool component, describe its requirements — the assembler decides implementation format:

- **Reusability** — used across campaigns, or campaign-specific?
- **Output contract** — structured JSON, or flexible format?
- **Consumers** — what skills/agents will use this tool's output?
- **Kit registration** — durable enough to register, or too ephemeral?

The planner describes the capability needed. The assembler picks the right template and implementation format.

---

## Tier 4: Orchestrated Investigation

Multiple tools, multiple rounds, judgment between rounds. Hypothesis-driven investigation loops.

| Aspect | Detail |
|--------|--------|
| Requirements | State tracking, hypothesis loop, multiple tools, reflection points |
| Infrastructure | Orchestrator, mental model tracking, tool access, potentially Docker |
| AI role | Investigator — drives strategy, forms hypotheses, adapts based on findings |
| Human role | Mission + approval gates |
| Output artifact | Agent definition + orchestration config |
| Example | "What's exposed on this org's perimeter" — agent iterates through discovery, mapping, probing |

**When to use:** The answer is not knowable in advance. You need adaptive multi-step reasoning where each step informs the next.

---

## Tier 5: Autonomous Multi-Agent

Event-triggered, multi-agent, runs without the practitioner. AI makes judgment calls at defined gates, deterministic enrichment runs in parallel.

| Aspect | Detail |
|--------|--------|
| Requirements | Event triggers, multi-agent orchestration, agent handoffs, data contracts |
| Infrastructure | Orchestrating agent + component agents, skills per agent, tools, monitoring |
| AI role | Multiple specialized agents at defined gates |
| Human role | Configuration + monitoring |
| Output artifact | Orchestrating agent (command or Agent SDK app) + component agent definitions + skills + tools |
| Example | CVE drops -> analysis -> triage -> research -> detection -> validated output |

**When to use:** The workflow is repeatable, gates are well-defined, and you want it running without you.

**Two execution modes for Pattern 3:**
- **3A (prompt-layer):** A command that chains Skill() calls -- runs in Claude Code. Registers as Kit type `command`.
- **3B (code-layer):** An Agent SDK application -- runs standalone. Built via `agent-sdk-dev:new-sdk-app` skill or Forge work orders. Registers as Kit type `tool`. Backlogged to flux.

---

## Tier Infrastructure Mapping

| Tier | Maps to | Exists? |
|------|---------|---------|
| 1 | Work order (Sable work system) | Yes |
| 2 | Skill (skill infrastructure) | Yes |
| 3 | Tool/wrapper + skill | Examples exist (Sigil helpers) |
| 4 | Orchestrated agent (Artemis pattern) | Yes |
| 5 | Multi-agent orchestration (Sigil architecture) | Partially |

## Kit Relationship by Tier

| Tier | What the catalog provides |
|------|--------------------------|
| 1 | Nothing — model uses training knowledge |
| 2 | Methodology skill selection |
| 3 | Wrapper tools + data contracts |
| 4 | Skills + wrappers + orchestration pattern |
| 5 | Agent definitions + skills + tools + orchestrator (command or Agent SDK app) |

---

## Tier Determination Rubric

| Signal | Tier 1 | Tier 2 | Tier 3 | Tier 4 | Tier 5 |
|--------|--------|--------|--------|--------|--------|
| Scope | Singular, known | Known approach, judgment in execution | Mechanical + interpretation | Multi-step adaptive | Repeatable pipeline |
| Judgment | None needed | Within guardrails | After deterministic output | Strategic throughout | At defined gates |
| Repeatability | One-off | Reusable methodology | Reusable mechanics | Reusable investigation | Event-triggered |
| Who drives | Practitioner directs | Intent + review | Intent only | Mission + approval gates | Config + monitoring |
| Time horizon | Minutes | Session | Session | Hours/days | Ongoing |

Disqualifying signals (cap the tier): "I know exactly what I want" → Tier 1. High-stakes novel target → cap at Tier 2-3. First time doing this work → cap at Tier 2.

Promoting signals (raise the tier): Output of step N changes step N+1 → Tier 4. "Run every time X happens" → Tier 5. Requires ordering/prioritization decisions → Tier 2+.

---

## Planner Output by Tier

| Tier | Planner produces |
|------|-----------------|
| 1 | Work order -> direct execution |
| 2 | Skill definition -> load and run |
| 3 | Plan with wrapper specs -> assembler builds, then run |
| 4 | Investigation plan -> assembler composes agent, then run |
| 5 | Orchestration spec -> assembler builds orchestrating agent + component agents + skills |

## Routing

- Tier 1-2: plan is simple enough -> execute directly (or via existing work system)
- Tier 3+: plan needs composition -> assembler skill runs -> produces runnable artifact
