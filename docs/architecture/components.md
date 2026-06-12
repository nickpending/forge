---
type: architecture
subtype: components
project: "forge"
status: active
created: "2026-03-24"
updated: "2026-06-12"
tags: [architecture, components]
---

# Components

Registry of all system components. Each entry links to a detail doc when the component has enough substance to warrant one.

## forge-strategist

**Purpose:** Security strategy consultant. Receives a sensemaker decomposition (already-resolved Key Decisions + practitioner overrides), builds a complete CONOPS, investigating only the specifics the decomposition didn't ground. Forked skill with hacker agent persona on opus model.
**Key files:** `~/development/projects/forge/skills/forge-strategist/SKILL.md` (forked, agent: hacker, model: opus), `~/development/projects/forge/skills/forge-strategist/references/` — nine reference files loaded before composing: forge-philosophy.md, forge-artifacts.md, forge-timing.md, conops-schema.md, context-sample.json, plus the composition model added 2026-06-04: composition-routing.md (five composition shapes), substrate-catalog.md (three substrates), agent-class-taxonomy.md (5 role + 7 skill base classes), extension-eval-protocol.md (Anthropic eval framework adapted to Forge), data-dependencies-schema.md (five-dimension data-dependency capture).
**Connections:** Reads the sensemaker decomposition report (`SENSEMAKER_REPORT` path) + `PRACTITIONER_OVERRIDES` + raw ask + context path (passed via $ARGUMENTS from the /forge campaign workflow, Stage 4). Does NOT re-litigate Key Decisions the practitioner already approved. Reads/writes `~/.config/forge/context.json`. Writes CONOPS to `~/.local/share/forge/conops/{slug}.md` with status `draft`. Emits `artifact_type_candidate`, `runtime_candidate`, `complexity_rationale`, and the new composition fields — `substrate`, `harness_weight`, `composition_rationale` — in CONOPS frontmatter, plus a `## Data Dependencies` section with a Reachable/Blocked feasibility verdict. Does NOT converse — runs autonomously and returns CONOPS path.
**Detail:** Not yet warranted — strategist rearchitected 2026-06-04 (ferret→sensemaker input swap; composition/substrate/agent-class model layered onto the artifact×runtime axes). New model is strategist-scoped; forge-planner and forge-assembler do not yet consume it.

## forge-planner

**Purpose:** Mechanical plan producer. Reads an approved CONOPS and forge context, determines artifact type + runtime via decision tree, produces a structured plan document per plan-schema.md v2.0. Forked skill — no conversation.
**Key files:** `~/development/projects/forge/skills/forge-planner/SKILL.md` (forked, context: fork), `~/development/projects/forge/skills/forge-planner/references/` (plan-schema.md v2.0, forge-patterns.md with six artifact types, kit-integration.md with five Kit types)
**Connections:** Reads approved CONOPS from `~/.local/share/forge/conops/`. Validates `artifact_type_candidate` and `runtime_candidate` from CONOPS. Queries Kit for available components. Classifies each component's reusability (reusable / parent-scoped / private). Writes plans to `~/development/projects/forge-armory/plans/`. Does NOT converse.
**Detail:** [components/forge-planner.md](components/forge-planner.md) *(needs update — rewritten from Pattern 0 inline to Pattern 1 forked)*

## forge-assembler

**Purpose:** Artifact-type-dispatched assembler that generates artifacts from plan documents, runs 3-level verification (structural → reference → execution), commits to armory, registers Kit-eligible artifacts only (five types: skill, tool, agent, command, harness), installs to XDG paths. Loads foundations before generation. Applies reusability rubric (reusable / parent-scoped / private) and bundle tag taxonomy per component.
**Key files:** `~/development/projects/forge/skills/forge-assembler/SKILL.md` (13-step workflow including Step 6.5: Level 3 execution testing), `~/development/projects/forge/skills/forge-assembler/references/` (composition-rules.md with 15 rules, verification-checklist.md, forge-runtime.md, tool-standards.md), `~/development/projects/forge/skills/forge-assembler/references/artifact-templates/` (7 artifact-type templates: skill.md, tool.md, agent-persona.md, agent-skills.md, command.md, automation-config.md, harness.md)
**Connections:** Reads plans from forge-armory/plans/. Writes artifacts to forge-armory/{type}/. Loads foundations via `Skill("prompt-foundations")`, `Skill("skill-foundations")`, `Skill("command-foundations")`. Runs verification per verification-checklist.md. Registers via `kit add` with campaign tags. Installs via `kit use`. Generates default config per forge-runtime.md. Updates `~/.config/forge/context.json` with campaign entries.
**Detail:** [components/forge-assembler.md](components/forge-assembler.md) *(needs update — manifests removed, foundations added, runtime contract added)*

## forge skill (entry point)

**Purpose:** Workflow router for forge operations. Dispatches to init, improve, or campaign workflows.
**Key files:** `~/development/projects/forge/skills/forge/SKILL.md` (routing table), `~/development/projects/forge/skills/forge/workflows/init.md` (runtime bootstrap), `~/development/projects/forge/skills/forge/workflows/campaign.md` (10-stage campaign pipeline), `~/development/projects/forge/skills/forge/workflows/improve.md` (artifact improvement flow)
**Connections:** Campaign workflow spawns ferret, forge-strategist, hacker, forge-planner, forge-assembler. Improve workflow reads artifacts from forge-armory, loads forge references from installed skills, presents changes for practitioner approval, commits back to forge-armory. Init workflow bootstraps forge runtime XDG skeleton.
**Detail:** Promoted from command to skill on 2026-04-08. Previously `commands/forge.md` — outgrew command pattern (had argument dispatch, multi-stage orchestration, sub-skill invocation).

## forge-armory

**Purpose:** Git repo storing ALL generated security artifacts — Kit-eligible and armory-only alike. Kit registers a subset (skill/tool/agent/command); automation configs co-locate with their tools.
**Key files:** `~/development/projects/forge-armory/` — skills/, agents/, tools/ (includes automation configs alongside tool binaries), plans/, commands/, harnesses/ (new — Agent SDK projects)
**Connections:** Written to by forge-assembler. Plans written by forge-planner. Read by Kit (artifacts registered via `kit add` point here). Cloned from `git@github.com:nickpending/forge-armory.git` (private).
**Detail:** [components/forge-armory.md](components/forge-armory.md)

## kit-catalog

**Purpose:** Kit component registry backing store — YAML catalog in its own git repo.
**Key files:** `~/development/projects/kit-catalog/kit-catalog.yaml`
**Connections:** Managed by Kit CLI (`kit add`, `kit list`, `kit sync`). Cloned to `~/.cache/kit/catalog/` on init. Remote at `git@github.com:nickpending/kit-catalog.git` (private). Currently 16 entries (10 workshop + 4 forge + 2 forge-armory).
**Detail:** [components/kit-catalog.md](components/kit-catalog.md)

## forge runtime

**Purpose:** Shared XDG infrastructure for configuration, state, and run memory across all forge-generated artifacts.
**Key files:** `~/.config/forge/context.json` (shared environment profile), `~/.config/forge/{thing}/config.json` (per-tool config), `~/.local/share/forge/{thing}/ledger.jsonl` (per-tool run history), `~/.local/share/forge/conops/` (CONOPS documents)
**Connections:** context.json read by all forge artifacts, written by strategist + assembler only. Per-tool config generated by assembler with first-run confirmation lifecycle. Defined in `forge-runtime.md` reference doc. `/forge init` bootstraps the skeleton.
**Detail:** Not yet warranted — contract is defined in forge-runtime.md.

## CONOPS (artifact type)

**Purpose:** Concept of Operations document — the interface between forge-strategist and forge-planner.
**Key files:** Schema at `~/development/projects/forge/skills/forge-strategist/references/conops-schema.md`. Instances at `~/.local/share/forge/conops/{slug}.md`.
**Connections:** Produced by forge-strategist. Pressure-tested by hacker agent. Approved by practitioner via /forge command. Consumed by forge-planner (sole input). Sections: Intent, Target, Scope, Flow, Data Dependencies (added 2026-06-04, with Reachable/Blocked feasibility verdict — a Blocked verdict halts CONOPS approval), Phases, Gotchas, Open Assumptions, Practitioner Context, Pressure Test Findings. Frontmatter gained `substrate`, `harness_weight`, and `composition_rationale` (conditional on artifact type) alongside the existing `artifact_type_candidate` / `runtime_candidate` / `complexity_rationale`.
**Detail:** Not yet warranted — schema doc is authoritative.

## composition model (strategist references)

**Purpose:** Strategist-scoped decision framework layered onto the artifact_type × runtime axes (2026-06-04). Picks the lowest-cost composition shape + substrate for an intent, and governs how agent specialists are composed and eval-validated.
**Key files:** `~/development/projects/forge/skills/forge-strategist/references/composition-routing.md` (five composition shapes: skill-wrapping-tool, automation-config, command, lightweight harness, heavyweight harness — plus the routing rubric), `substrate-catalog.md` (three substrates: CC CLI [real], Python orchestrator [to build, Q3 2026], Anthropic Agent SDK [real] — each with a slot contract), `agent-class-taxonomy.md` (12 base classes: 5 roles — Auditor/Debater/Prover/Communicator/Detector — + 7 skill bases; specialist = one role base + N skill bases + specialization + context-pack; `{role}::{scope}::{specialty}` naming), `extension-eval-protocol.md` (capability + regression evals gate base-class promotion and specialist Kit registration; Anthropic eval vocabulary), `data-dependencies-schema.md` (five-dimension capture: location, access, ownership, curation, runtime accessibility).
**Connections:** Read by forge-strategist during CONOPS construction (Step 1 loads all of them). composition-routing.md output → CONOPS `substrate` / `harness_weight` / `composition_rationale` frontmatter. data-dependencies-schema.md output → CONOPS `## Data Dependencies` section, also consumed by forge-planner per the schema doc. extension-eval-protocol.md is the gate forge-assembler must honor before promoting a specialist into Kit. NOT yet referenced by the planner or assembler SKILL.md files — model lives entirely in strategist references for now.
**Detail:** Not yet warranted — the five reference docs are authoritative. Uncommitted as of 2026-06-12 baseline (working-tree changes dated 2026-06-04).
