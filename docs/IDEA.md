---
type: project
domain: technical
status: active
started: 2026-03-20
---
# Forge - AI-Native Security Practice Framework

## The Problem

**What specific problem does this solve?**

Security practitioners with AI capabilities face a decision problem: when to run tools directly, when to write scripts, when to build skills, when to compose agents, when to stand up pipelines. A year of building security projects (ASH, Artemis, Sigil, Arsenl, mcp-recon) produced foundational ideas and working prototypes, but nothing material for running actual campaigns today. The gap isn't technical capability — it's a missing philosophy and operational framework for how security work should be conducted when AI is a first-class participant.

**Who has this problem?**

Security practitioners (from junior analysts to senior operators) who work with AI-assisted tooling and need a systematic way to plan, compose, and execute security work across the full spectrum — from quick port scans to multi-agent autonomous pipelines.

**How do they solve it today?**

Ad-hoc decisions. Every task starts from scratch: "should I just run nmap, or build a skill, or write a script?" No framework for deciding. No reusable catalog of capabilities. No way to compose tested components into campaigns. Practitioners either under-engineer (run everything manually) or over-engineer (build an agent when a command would do).

## The Solution

**Core Value Proposition**

Forge takes security intent and produces operational capability — the right artifact at the right level of AI involvement, calibrated to the practitioner and the task. A planner skill (`/forge`) determines the approach, an assembler composes validated artifacts, and a catalog tracks reusable components.

**Key Differentiators**

- Philosophy-driven: five principles govern when AI adds value vs when it adds latency
- Tier-aware: five tiers of AI involvement (direct tool use → autonomous pipeline), each with clear infrastructure requirements
- Pattern-based: four artifact patterns (inline skill, forked skill-agent, first-class agent, orchestrator) matched to work shape
- Practitioner-calibrated: adapts output scaffolding to operator experience level without reducing rigor
- Validated: assembler proves campaigns work in Docker before handoff
- Proactive: planner proposes campaigns from available datasets/access, not just reactive to requests

## System Flow (Initial Sketch)

> **Note**: This is a preliminary sketch of system operation. The actual workflow will evolve significantly during development.

1. Practitioner invokes `/forge` with security intent (or planner proactively proposes based on available data)
2. Planner determines tier (1-5), artifact pattern (0-3), and execution mode (interactive/automated/scheduled)
3. For Tier 1-2: planner produces plan/skill directly → execute
4. For Tier 3+: planner produces plan → assembler selects from catalog, composes artifact, validates in Docker
5. Validated artifact executes in selected mode, producing layered results + operator logs

## User Experience Vision

**Primary User Journey**

1. `/forge` — describe what you want to accomplish ("check this censys dump for exposed HTTP services")
2. Planner engages: clarifies scope, determines tier, adapts to practitioner level
3. Output: the right artifact for the job (plan, skill, agent definition, or pipeline spec) — ready to run

**Core User Workflows**

- **Plan a campaign** — describe intent, get structured methodology with tier/pattern recommendation
- **Compose a campaign** — planner + assembler produce validated, runnable artifacts from catalog components
- **Run a campaign** — execute composed artifacts interactively, automated, or on schedule
- **Review results** — layered output (structured data for models, narrative for humans) + operator logs
- **Grow the catalog** — successful campaigns contribute components back to the catalog

**Success Criteria**

- Can go from "I have this dataset/target" to "running a campaign" in under 30 minutes
- Planner produces plans that a junior analyst can follow to achieve senior-level results
- Assembled campaigns are validated before execution — no "it should work" handoffs
- Catalog grows organically from successful work — components become reusable

## MVP Definition

**What is the absolute minimum viable version?**

The planner skill (`/forge`) as a Pattern 0 inline skill that takes security intent and produces structured plans. No assembler, no catalog utility, no Docker validation. Just the planner, practitioner expertise, and structured methodology output. This alone gets from "I have ideas but nothing material" to "I have a plan I can follow manually."

**MVP Scope**

- `/forge` planner skill (Pattern 0, inline, main thread)
- Tier determination logic (assess work shape → recommend tier)
- Structured plan output calibrated to practitioner level
- Integration with existing Sable work system for Tier 1-2 execution

**MVP Constraints**

- No assembler — plans are followed manually or fed to existing work system
- No Kitintegration — component inventory is ad-hoc until @voidwire/kit exists
- No Docker validation — trust the plan
- No agent personas — just plans and methodology
- Pattern 0 only — no forked agents or orchestrators yet

**Post-MVP Evolution**

- Stage 1: Build 2-3 concrete security skills + seed the catalog
- Stage 2: Assembler skill for Tier 3+ composition with Docker validation
- Stage 3: Agent personas, execution modes (automated/scheduled), operator log framework
- Stage 4: Kitintegration (depends on @voidwire/kit from llmcli-tools), cross-session learning from results

## Features Status

**Status Legend:**

- 📋 **Planned** - Feature defined and ready for iteration planning
- 🔄 **In Progress** - Feature currently being developed (iteration-N)
- ✅ **Built** - Feature completed and shipped

**Current Features:**

- 📋 **Planner Skill (`/forge`)** — Pattern 0 inline skill, universal entry point for security work
- 📋 **Tier Framework** — Five tiers of AI involvement with infrastructure mapping
- 📋 **Artifact Patterns** — Four patterns (0-3) for different work shapes
- 📋 **Assembler Skill** — Tier 3+ composition from catalog with Docker validation
- 📋 **KitIntegration** — Queries `@voidwire/kit` (cross-cutting component registry) for available skills/agents/wrappers
- 📋 **Agent Personas** — Security-specific personas with people/code names
- 📋 **Execution Modes** — Interactive, automated, and scheduled campaign execution
- 📋 **Operator Logs** — Cross-cutting reasoning trace observability
- 📋 **Layered Results** — Structured core + narrative layer, density by tier
- 📋 **Proactive Planning** — Campaign proposals from available datasets/context

## Technical Approach

**Architecture Decision**

- [x] **Composed Tool Ecosystem** - Multiple components with clean interfaces

**Why this approach?**

Forge is a philosophy + a set of skills + a catalog utility — not a monolithic application. The planner and assembler are skills that run on Sable's infrastructure. The catalog is a TypeScript utility that manages an index. Agent personas are markdown files. Everything composes with existing infrastructure.

**Dependencies & Prerequisites**

- Sable runtime (skills, hooks, work system)
- Claude Code (skill infrastructure, agent types, forked skill execution)
- `@voidwire/kit` — cross-cutting component registry (in llmcli-tools monorepo)
- Docker (for assembler validation at Tier 3+)
- Security tools installed on host (nmap, nuclei, httpx, etc. as needed)

**Integration Requirements**

- Sable skill system — planner and assembler are Claude Code skills
- Sable work system — Tier 1-2 work routes through existing orchestration
- Lore — cross-session learning from campaign results
- Existing security projects — Artemis patterns for Tier 4, Sigil patterns for Tier 5

**Key Technical Constraints**

- Skills must follow Sable skill architecture (frontmatter, markdown instructions)
- Forked skills require `context: fork` + `agent:` field referencing a persona
- Agent personas follow identity/instructions separation (WHO vs WHAT TO DO)
- Catalog entries need sufficient metadata for assembler selection

## Technical Architecture (Tentative)

> **Note**: This section captures current technical thinking and design exploration. All architectural decisions and implementation details are subject to change.

**Component Architecture (Working Model)**

```
/forge (planner skill)           Pattern 0, inline, main thread
    ↓
  Tier 1-2: direct execution or existing work system
  Tier 3+:  → assembler skill
                ↓
              catalog utility → select components
                ↓
              compose artifact (skill, agent, orchestrator)
                ↓
              Docker validation
                ↓
              runnable campaign artifact
```

**Artifact Pattern Architecture**

| Pattern | Type | Frontmatter | Execution |
|---------|------|-------------|-----------|
| 0 | Inline skill | `name`, `description` | Main thread |
| 1 | Forked skill-agent | `context: fork`, `agent: <persona>`, `model: <model>`, `allowed-tools: [...]` | Subprocess |
| 2 | First-class agent + skills | Agent persona definition + Pattern 0 skill references | Agent SDK or forked |
| 3 | Multi-agent orchestrator | Pipeline definition + agent sequence + data contracts | Command / standalone |

**Where Things Live**

| Component | Location | Format |
|-----------|----------|--------|
| Planner skill | Sable skills directory | SKILL.md |
| Assembler skill | Sable skills directory | SKILL.md |
| Kitregistry | llmcli-tools monorepo (@voidwire/kit) | CLI tool (cross-cutting) |
| Agent personas | Forge project | Markdown files |
| Generated artifacts | Per-campaign output directory | Skills, scripts, configs |
| Operator logs | Per-campaign output directory | Structured markdown |

## Implementation Strategy (Subject to Change)

> **Note**: These represent current thinking and will evolve.

**Iteration Priorities (Draft)**

1. **Stage 0 (MVP):** `/forge` planner skill — structured plans from intent
2. **Stage 1:** 2-3 concrete security skills + catalog seed
3. **Stage 2:** Assembler skill with catalog selection and Docker validation
4. **Stage 3:** Execution modes + operator log framework + agent personas
5. **Stage 4:** Catalog utility (TypeScript) + cross-session learning

**Validation Points**

- After Stage 0: "Can I go from intent to a plan I can follow manually?"
- After Stage 1: "Does `/forge` reference real components in its plans?"
- After Stage 2: "Can the assembler produce a validated, runnable campaign?"
- After Stage 3: "Can I fire-and-forget a campaign and review results later?"

## Learning and Evolution

**Key Learnings (Pre-Build)**

- A year of security project exploration (ASH → Artemis → Sigil → Arsenl) proved individual patterns but didn't produce usable day-to-day tooling
- The gap was philosophy and framework, not technical capability
- Skills replace command catalogs for methodology — but grounding data (arsenl's insight) still has value as a planning accelerator
- Forked skills ARE agents — the `context: fork` + `agent:` frontmatter pattern collapses the skill/agent distinction
- mcp-recon's MCP-based approach is obsolete given skills, hooks, and agent SDK

**Evolution Notes**

- Originated from a discussion about arsenl's merits and how to conduct recon/campaigns
- Evolved from project-level analysis to a full philosophy of AI-native security practice
- Named "Forge" — takes raw security intent and shapes it into operational capability
- The philosophy (principles, tiers) came first; the operational framework (planner, assembler, catalog) fell out of it
- Full exploration: [[reference/technical/explorations/security-work-philosophy|Forge: AI-Native Security Practice]]

## Open Questions

**Technical Questions**

- Kitintegration — how does Forge query @voidwire/kit? CLI? Library import?
- Agent persona names — people names or code names for security agent roles
- Cross-session learning mechanism — Lore captures? Kitannotations? Both?
- Forge lives at ~/development/projects/forge/ — standalone project, `forge init` installs skills

**Design Questions**

- Practitioner calibration mechanism — how does the planner know experience level?
- Relationship to `/orchestrate-work` — does `/forge` invoke it for Tier 1-2, or produce standalone artifacts?
- Results schema — how prescriptive should the structured core be?
- How tightly does arsenl's tool intelligence integrate with the catalog?

## Success Metrics

**Primary Metrics**

- Time from security intent to running campaign (target: <30 minutes for Tier 1-3)
- Practitioner can follow planner output without external reference material
- Assembled campaigns execute successfully on first run (Docker-validated)

**Learning Metrics**

- Catalog growth rate — are successful campaigns producing reusable components?
- Tier distribution — are practitioners naturally using the right tier for the right work?
- Plan quality — do junior analysts following plans produce comparable results to senior operators?

## Risks and Assumptions

**Key Assumptions**

- Security practitioners will adopt a structured planning step before execution (some may resist)
- The planner can encode enough security methodology to produce useful plans from intent alone
- Sable's skill infrastructure is sufficient for the planner and assembler (no Agent SDK required for MVP)
- The tier framework is intuitive enough that practitioners will internalize it

**Primary Risks**

- Over-engineering: building the catalog and assembler before the planner proves valuable
- Philosophy without practice: the exploration doc is comprehensive but no code exists yet
- Scope creep: Forge touches everything (skills, agents, pipelines, catalogs) — easy to lose focus
- Naming collision: agent personas must not conflict with existing Sable agent types

**Mitigation Strategies**

- Stage 0 first: prove the planner delivers value before building anything else
- Real campaigns: use Forge for actual security work immediately, not hypothetical examples
- Tight staging: each stage must produce usable output, not just infrastructure
