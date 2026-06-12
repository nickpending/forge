---
type: architecture
subtype: overview
project: "forge"
status: active
created: "2026-03-24"
updated: "2026-06-12"
tags: [architecture]
---

# Forge Architecture

Forge is an AI-native security practice framework that takes security intent and produces operational capability — the right artifact at the right level of AI involvement. A sensemaker agent decomposes intent into ranked Key Decisions; a forked strategist turns the approved decomposition into a CONOPS, selecting an artifact type, runtime, composition shape, and substrate; a planner skill produces a mechanical plan from the CONOPS. An assembler composes validated artifacts, execution-tests them (Level 3), and commits to the armory. The armory stores all artifact types; Kit registers only the AI-layer subset (skill/tool/agent/command/harness). Automation configs (Justfiles, cron, n8n workflows) co-locate with their tools and run without model involvement.

The strategist works against a composition model (2026-06-04, strategist-scoped): five composition shapes selected across three runtime substrates (CC CLI, Python orchestrator, Agent SDK), with agent specialists composed from a 12-class base taxonomy and gated by an extension eval protocol before they enter Kit. The artifact_type × runtime axes remain primary; composition shape and substrate are how an agent-driven campaign is actually realized.

## Principles

- **Philosophy-driven**: Five principles govern when AI adds value vs when it adds latency
- **Tier-aware**: Five tiers of AI involvement (direct tool use → autonomous pipeline), each with clear infrastructure requirements
- **Pattern-based**: Four artifact patterns (inline skill, forked skill-agent, first-class agent, orchestrator) matched to work shape
- **Practitioner-calibrated**: Adapts output scaffolding to operator experience level without reducing rigor
- **Validated**: Assembler proves campaigns work before handoff

## Components

| Component | Purpose | Detail |
|-----------|---------|--------|
| forge-strategist | Builds a CONOPS from a sensemaker decomposition; applies the composition-routing / substrate / agent-class model | [components.md](components.md#forge-strategist) |
| forge-planner | Determines artifact type, runtime, and plan from an approved CONOPS | [components.md](components.md#forge-planner) |
| forge-assembler | Composes validated artifacts from plan documents | [components.md](components.md#forge-assembler) |
| forge-armory | Git repo storing generated artifacts by type | [components/forge-armory.md](components/forge-armory.md) |
| kit-catalog | Component registry catalog (Kit backing store) | [components/kit-catalog.md](components/kit-catalog.md) |
| forge skill | Entry point — routes to init, improve, or campaign workflows | `skills/forge/SKILL.md` (workflow router) |

## Key Decisions

See [decisions.md](decisions.md) for the full decision log.

## Boundaries

See [boundaries.md](boundaries.md) for interface contracts.
