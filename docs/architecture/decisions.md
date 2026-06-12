---
type: architecture
subtype: decisions
project: "forge"
status: active
created: "2026-03-24"
updated: "2026-06-12"
tags: [architecture, decisions]
---

# Decisions

Architectural decisions and their rationale. Most recent first.

## Composition model + sensemaker decomposition (2026-06-04, uncommitted as of baseline)

**Context:** The artifact_type × runtime model answered "what to build and where it runs" but left the strategist without a rubric for *how to compose* a campaign from existing parts, *which runtime substrate* hosts an agent-driven campaign, or *how agent specialists are built and trusted*. The strategist also still consumed ferret's text-only disambiguation, which decomposed intent more weakly than a dedicated decomposition agent. This decision is grounded in the strategist rewrite (`skills/forge-strategist/SKILL.md`) and five new reference files (`composition-routing.md`, `substrate-catalog.md`, `agent-class-taxonomy.md`, `extension-eval-protocol.md`, `data-dependencies-schema.md`), all dated 2026-06-04 in the working tree.
**Choice:** Two moves. (1) **Composition model** layered onto the existing axes, strategist-scoped: five composition shapes (skill-wrapping-tool, automation-config, command, lightweight harness, heavyweight harness) selected against three substrates (CC CLI [real], Python orchestrator [to build, Q3 2026], Anthropic Agent SDK [real]); agent campaigns compose a specialist as `one role base + N skill bases + specialization + context-pack` from a locked 12-class seed (5 roles: Auditor/Debater/Prover/Communicator/Detector + 7 skill bases), named `{role}::{scope}::{specialty}`; base-class promotion and specialist Kit registration are gated by an extension eval protocol (capability + regression evals, Anthropic vocabulary: Task/Trial/Grader/Transcript/Outcome). CONOPS gains `substrate`, `harness_weight`, `composition_rationale` frontmatter (conditional on artifact type) and a required `## Data Dependencies` section (five dimensions; a Blocked feasibility verdict halts CONOPS approval). (2) **Sensemaker replaces ferret** as the pre-strategist step: the campaign workflow Stage 2 spawns the `sensemaker` agent (Iris Novak persona) to produce a 5-section decomposition (Synthesized Intent, Key Decisions, Flagged Gaps, Artifact Sources, Scope Boundaries) with ranked resolutions; the strategist receives `SENSEMAKER_REPORT` + `PRACTITIONER_OVERRIDES` + raw ask and does NOT re-litigate approved Key Decisions.
**Why:** "Compose what exists, build what doesn't" needs an explicit rubric or the model defaults to building from scratch; the composition-routing rubric forces a check for an existing tool/substrate/base-class first. Substrates separate "where the agent runs" from "what artifact type it is" — the same agent runs on CC CLI, a Python orchestrator, or the Agent SDK. The eval protocol prevents untrusted specialists entering Kit (parallels the existing Level 3 testing gate for tools). Data dependencies catch the failure mode where a campaign is proposed that can't run because its data isn't reachable from the chosen substrate. Sensemaker decomposition is stronger than ferret disambiguation and lets the strategist do lighter, non-conversational work from an already-concrete intent. **Scope note:** the model is strategist-scoped — forge-planner and forge-assembler SKILL.md do not yet consume it (confirmed: no `substrate` / `composition-routing` / `agent-class` references in their skills as of baseline).

## Artifact type × runtime replaces tier-as-autonomy (2026-04-16)

**Context:** Investigating the gray zone between automations, human-augmenting skills, and autonomous agents revealed that forge's 1-5 tier integer conflated artifact shape, runtime, and scope/complexity rubric into one number. Trail of Bits' skills repo (all category-b: human-session methodology packages), Prismis corpus on LLM vulnerability discovery, and the mature Claude Code bug-bounty ecosystem (shuvonsec, transilience, Orizon) confirmed that the real axes are artifact type (what you build) and runtime (where it runs). External adversarial security was the first serious customer, not the box.
**Choice:** Primary axis is now `artifact_type` (six types: tool, skill, agent, command, automation_config, harness) × `runtime` (four: human_session, scheduled_claude_code, agent_sdk_runtime, deterministic_pipeline). Tier demoted to `complexity_score` + `complexity_rationale` — used for scaffolding density, not routing. Plan schema v2.0. Assembler templates renamed from pattern-prefixed to artifact-type-named. Campaign routing changed from tier-integer to components_needed-based. Kit registers five types (added harness). Reusability rubric (reusable / parent-scoped / private) governs inner artifacts of composites, with `bundle:{parent-type}:{parent-slug}` tag taxonomy. Skill invocation modes (inline / forked) are a constraint classifier with assembler-time lint for four interaction-marker categories.
**Why:** The tier conflation caused the planner to cascade through tier → pattern → runner when these should be direct selections. Same skill runs in a human session, a scheduled cron, or inside an Agent SDK harness — the artifact doesn't change, the runtime does. Harness (Agent SDK app with LLM-driven control flow) is genuinely a different artifact type from command (in-Claude-Code orchestrator) and automation_config (deterministic pipeline). The orchestrator trilemma — "who decides the next step" — is the forcing question. See exploration: `obsidian/reference/technical/explorations/forge/artifact-and-runtime-model.md`.

## Operational capability model — tiered AI involvement with runner mapping (superseded by above)

**Context:** Running http-recon revealed forge produces artifacts designed to be model-executed, but many should be automation-executed (cron, n8n, Justfile). Trail of Bits' skills repo confirmed the pattern — their plugins are all AI methodology, none wrap long-running tools. Forge needed to encode the distinction between AI-driven and automation-driven work.
**Choice:** Each forge tier now maps to a runner type (direct execution, Claude Code, Justfile/cron/n8n, Agent SDK). The planner recommends both tier AND runner. Plan schema includes `runner`, `kit_eligible_components`, and `armory_only_components` fields. Assembler produces all artifacts but only calls `kit add` for Kit-eligible types (skill/tool/agent/command). Automation configs co-locate with their tool in `tools/{name}/`.
**Why:** The model's value is in planning campaigns and analyzing results, not babysitting execution. Deterministic tools should run via cron/n8n/Justfile. Skills should analyze output, not wrap execution. This separation also enables mandatory artifact testing (Level 3) — tools get execution-tested, Justfiles get dry-run, skills get structural validation only.

## Skill-scoped hooks for safety gates

**Context:** Trail of Bits plugins use hooks for input validation, command interception, and quality gates. Forge needed hook awareness but uses Kit's type system (skill/command/tool/agent), not the Claude Code plugin system.
**Choice:** Forge uses skill-scoped hooks in SKILL.md frontmatter, not plugin-level hooks. Two patterns: PreToolUse command hooks (shell scripts — fast, deterministic, for scope checking and rate limiting) and Stop prompt hooks (model-evaluated, for quality gates). Hook scripts live in the skill's `tools/` directory. Hooks require a consuming skill — tool-only Tier 3 artifacts without a consuming skill cannot have hooks.
**Why:** Skill-scoped hooks auto-cleanup when the skill is done, travel with the skill via Kit, and don't require the plugin system. The shell/prompt split matches the deterministic/probabilistic boundary from forge-philosophy.md Principle 2.

## Automation configs co-locate with tools, not in separate directory

**Context:** Needed a home for non-Kit artifacts (Justfiles, n8n workflows, cron configs) in the armory. Options: separate `automations/` directory, type-specific directories, or bundled with tools.
**Choice:** Automation configs live in `tools/{name}/` alongside the tool they automate. No separate `automations/` directory. Kit registers the tool binary only; configs are co-located but not Kit-registered.
**Why:** Automation configs are useless without their tool — they depend on the tool binary. Co-location keeps related artifacts together. The tool is standalone (Kit installs it to `~/.local/bin/`); the configs are optional companions for recurring/scheduled execution.

## Mandatory Level 3 execution testing before armory commit

**Context:** Forge assembler could commit untested artifacts to the armory. No verification that a generated tool actually runs.
**Choice:** New Step 6.5 in assembler workflow: tools get execution-tested (run with derived test inputs, verify exit code + JSON output), Justfiles get `just --dry-run`, n8n workflows get JSON schema validation, skills/commands/agents get structural validation only (existing Level 1+2).
**Why:** Untested tools are liabilities, not deliverables. Level 3 inserts between Level 2 verification and armory commit — the artifact must prove it works before it gets shipped.

## Tool quality standards for assembler

**Context:** The assembler generates tools (probe.ts, http-filter, etc.) without quality guidance. Tools were built correctly by model luck, not encoded standards. The CLI development guide at `~/obsidian/reference/technical/development/cli-development-guide.md` covers the full standard but is too large to load into every assembler run.
**Choice:** Compressed `tool-standards.md` reference in assembler's references directory. Covers: Bun/TypeScript stack, library-first architecture, manual arg parsing, JSON output, zero dependencies preferred, strict types, JSON errors. Tools are built for AI assistants to operate.
**Why:** Assembler needs quality guardrails for tool generation without loading a 480-line guide. The compressed version captures the constraints that prevent violations (wrong stack, framework dependencies, plain text output) without the full tutorial content.

## Improve flow — AI judgment with tool change decision tree

**Context:** http-recon was built before timing profiles existed. No mechanism to update existing campaign artifacts when the forge pipeline evolves. First attempt proposed changing probe.ts defaults (architectural) alongside skill edits (mechanical) — needed a rubric to distinguish them.
**Choice:** New improve workflow (`/forge improve <artifact>`) uses AI judgment to diff artifacts against current forge references. Changes classified by layer using IF-THEN-EXAMPLES decision tree: skill-only (tool already accepts the flags), mechanical tool change (additive, follows existing pattern), or architectural (recommend only — alters defaults, restructures, affects other consumers). Practitioner approves per-change.
**Why:** AI judgment is simpler than structured manifests and catches all gap types. The decision tree prevents the model from making design decisions about tool architecture while allowing additive mechanical changes. Tested against http-recon — correctly classified 6 gaps across both layers on second run.

## Forge promoted from command to skill

**Context:** forge.md was a command acting as a skill — had argument dispatch (init + campaign pipeline), invoked sub-skills (strategist, planner, assembler), exceeded 200 lines. Adding the improve flow would make it worse. Skill-foundations Workflow Router archetype says split into routing table + workflow files when >200 lines.
**Choice:** Promote forge from `commands/forge.md` to `skills/forge/SKILL.md` with `workflows/` directory (init.md, campaign.md, improve.md). Kit re-registered as type: skill. Old command deleted.
**Why:** Commands are single markdown files with no routing, workflows, or references. Forge outgrew that. The skill pattern provides proper structure — routing table dispatches to workflow files, each workflow is self-contained. Also enabled the improve workflow to be added cleanly as a new route.

## Timing profiles as forge reference constants

**Context:** http-recon had hardcoded rate limits (50 rps / 25 threads in probe.ts). Operator's BGW620 router enforces 50 pps sustained on LAN but cellular has no constraint. Per-skill hardcoding doesn't scale — every new campaign would need to discover the same constraint.
**Choice:** Timing profiles (T1 cautious / T2 moderate / T3 aggressive) as a reference file in the strategist's references. The strategist reads it and writes values into the CONOPS. Planner reads CONOPS, assembler reads plan — data flows through pipeline documents, not cross-skill file reads. Profiles are constants, not interview questions or context.json fields.
**Why:** Abstracted levels (like nmap's T0-T5) let campaigns reference speed by name. The operator's specific network constraints determine which profile they select. The campaign doesn't need to know about the router — only that T1 means 40 pps / 25 threads.

## Forge runtime layer — shared XDG config/state for all generated artifacts

**Context:** Every forge-generated tool was inventing its own storage for config and run history. Resolvers, dataset paths, and environment data were duplicated per tool. No tool remembered what it did last run.
**Choice:** Forge runtime at XDG paths: `~/.config/forge/` for config (context.json shared, per-tool config.json), `~/.local/share/forge/` for state (per-tool ledger.jsonl, CONOPS). context.json is read-only for tools, writable by strategist + assembler only. `/forge init` bootstraps. Assembler bakes runtime preamble into every generated artifact.
**Why:** One source of truth for environment data. Tools read shared context instead of duplicating it. Per-tool config with first-run confirmation lifecycle. Ledger provides run memory without full logging.

## Forked strategist with hacker persona + ferret pre-processing

**Context:** Five live /forge test runs showed the inline Pattern 0 strategist fails — persona diluted by main thread, asks practitioner to do the thinking, needs 8+ turns to converge, makes implementation decisions. Ferret experiment showed text-only disambiguation in 20 seconds identified ambiguities the strategist missed in 8 turns.
**Choice:** forge-strategist becomes Pattern 1 forked with `agent: hacker` (Kira Voss persona) and `model: opus`. /forge command adds ferret pre-processing (Stage 2: text-only disambiguation, Stage 3: present to practitioner). Strategist receives ferret output + raw ask + context path, investigates autonomously, produces CONOPS in one shot.
**Why:** Forked context gives clean persona without main thread dilution. Hacker persona provides security domain reasoning. Opus model for strategic quality. Ferret resolves ambiguity before the strategist runs, giving it a clean input. /forge handles all practitioner interaction.

## Split planner into strategist + mechanical planner

**Context:** Monolithic forge-planner tried to be a conversational consultant AND a structured plan producer. Five live runs proved it oscillated between modes, doing neither well — asked lazy questions, rushed to tier determination, proposed implementation details.
**Choice:** forge-strategist (Pattern 1 forked, produces CONOPS) + forge-planner (Pattern 1 forked, reads CONOPS, produces plan per schema). CONOPS is the interface document. /forge command orchestrates the pipeline with approval gates.
**Why:** Two fundamentally different cognitive tasks: conversational exploration vs structured output production. Splitting them lets each component optimize for its job. CONOPS carries tier rationale with rubric signals so the planner validates without needing duplicate references.

## Bun single-file default for tools

**Context:** Pattern 1 tool template had 4-file npm structure (index.ts, cli.ts, package.json, tsconfig.json). Most forge-generated tools are < 500 lines with no external consumers or build pipeline.
**Choice:** Default to single-file Bun (shebang + import.meta.main guard). Multi-file npm pattern is the escalation for complex tools (> 500 LOC, external library consumers, npm dependencies, build pipeline needed).
**Why:** Bun runs TypeScript natively. A 200-line filter script doesn't need package.json and tsconfig.

## Fork/inline selection rule

**Context:** Assembler proposed forking skills that need user interaction. No rule prevented this.
**Choice:** Explicit rule in forge-patterns.md and composition-rules.md (Rule 12): forked skills have no conversation context, any skill requiring practitioner interaction MUST be Pattern 0 inline.
**Why:** Forked skills run in isolation. If a skill needs to ask questions, present options, or wait for approval, it must be inline. All inputs to a forked skill must be provided upfront.

## Kit manifests removed — kit add CLI only

**Context:** Assembler Step 8 wrote kit-manifest.yaml sidecar files that Kit never reads. Kit uses `kit add` CLI with flags, not manifest files.
**Choice:** Deleted Step 8 entirely. All templates updated: Kit Manifest → Kit Registration with `kit add` command. Campaign tagging: all artifacts share `campaign:{slug}` tag.
**Why:** Kit-manifest.yaml was fiction. Kit's catalog is centralized at a git repo, updated via CLI. Campaign grouping uses tags (`kit list --tags campaign:{slug}`).

## Tool vocabulary: wrapper is a subtype, not a Kit type

**Context:** Forge used "wrapper" as both an artifact type and directory name. Kit's types are `skill | command | tool | agent`. The planner had no guidance for choosing between tool subtypes (wrapper, CLI, script).
**Choice:** `wrappers/` → `tools/` everywhere (armory, assembler paths, kit manifests). Keep "wrapper" as a conceptual descriptor in composition rules (Rules 2 and 8 describe the design pattern). Add tool-subtype decision guidance to the planner grounded in Principle 2 (deterministic vs probabilistic).
**Why:** A wrapper is one kind of tool. Not every tool is a wrapper. Armory directories should match Kit types 1:1: `skills/`, `agents/`, `tools/`, `plans/`. The planner needs criteria for when to produce a wrapper vs standalone CLI vs quick script.

## Replace pipeline with Pattern 3A/3B orchestrator

**Context:** pattern3-pipeline.md generated standalone YAML that nothing could execute. Kit has no `pipeline` type. The template was inconsistent with every other pattern (all others produce markdown with frontmatter).
**Choice:** Replace with Pattern 3A (prompt-layer orchestrator command) and backlog Pattern 3B (Agent SDK app). The /forge command itself IS a Pattern 3A orchestrator — used as the reference example for the new template.
**Why:** Per security-work-philosophy.md, Pattern 3 has two execution modes: 3A chains Skill() calls in a markdown command (Kit type: `command`), 3B is an Agent SDK application (Kit type: `tool`, backlogged). No `pipeline` Kit type exists or is needed. Pattern 3A orchestrators produce multiple artifacts — the orchestrator registers as `command`, component agents and skills register individually.

## Kit type rename: script → tool

**Context:** Kit CLI is renaming its `script` ResourceType to `tool`. Forge referenced `--type script` in the assembler's Kit mapping table and used "script/scripts" vocabulary throughout tier descriptions, composition rules, and templates.
**Choice:** Rename all `script/scripts` vocabulary to `tool/tools` across forge. The package.json `"scripts"` field is excluded (standard npm, not a Kit type).
**Why:** Kit will search for `tool`, not `script`. If forge says `--type script`, the assembler's `kit add` calls break silently. Vocabulary alignment also prevents confusion in tier descriptions.

## Distributable artifacts use top-level type directories

**Context:** Task 6.1 initially placed `forge.md` at `.claude/commands/forge.md` following the `prime-*.md` pattern. User identified this was wrong — `.claude/` is reserved for project development configuration (settings.json, sable framework symlinks), not distributable artifacts.
**Choice:** Distributable artifacts live in top-level type directories at project root: `skills/`, `commands/`, `agents/`, `tools/`. Kit's catalog `path` field references these source locations; Kit handles mapping to install destinations (`~/.claude/commands/`, `~/.claude/skills/`, etc.).
**Why:** The `prime-*.md` files in `.claude/commands/` are sable framework symlinks installed for development, not project-owned distributable artifacts. Conflating the two means Kit would source from the wrong location and `.claude/` would accumulate artifacts that belong in the project's distributable namespace.

## ${CLAUDE_SKILL_DIR} for reference loading in forge skills

**Context:** forge-planner needs to load 6 bundled reference files at runtime. The plan flagged `$CLAUDE_SKILL_DIR` as a MEDIUM risk because no observed Sable skill used it for self-referencing. Investigation revealed it's a documented Claude Code string substitution.
**Choice:** Use `${CLAUDE_SKILL_DIR}/references/` for all reference loading. No inline duplication of reference content as fallback.
**Why:** `${CLAUDE_SKILL_DIR}` resolves dynamically to wherever the skill is installed — works in dev (`~/development/projects/forge/skills/`) and after `kit use` (`~/.claude/skills/`). Confirmed working in ideation/SKILL.md. References are authoritative; the skill orchestrates workflow.

## INV-005 gitignore guard as structural protection

**Context:** context.json is lazy-created by forge-planner on first invocation. File-absence checks pass trivially before that happens — they don't prove protection exists.
**Choice:** Treat the .gitignore exclusion of context.json as a separate named invariant (INV-005 precondition), tested independently from file absence.
**Why:** The gitignore guard must exist before the planner first runs. A broken gitignore discovered after first planner run would already be a data leak. Structural protection > incidental absence.

## Private repos for kit-catalog and forge-armory

**Context:** Plan originally specified all three repos as public. User overrode during build.
**Choice:** kit-catalog and forge-armory are private GitHub repos. forge remains public.
**Why:** Catalog contains metadata about security tooling inventory. Armory contains generated security artifacts. Neither should be publicly visible. The forge project itself (framework code) is public.
