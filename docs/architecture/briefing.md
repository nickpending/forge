Forge is a security practice framework that generates operational capabilities from security intent. It uses a planner to determine approach, an assembler to compose validated artifacts, and an armory to store them. Kit registers AI-layer artifacts.

Key data flow: Security intent → sensemaker (decomposition) → forge-strategist (CONOPS, with composition shape + substrate) → forge-planner (plan) → forge-assembler (artifacts) → forge-armory (storage) → Kit (registration). Automation configs co-locate with tools.

Contracts:
- Kit CLI ↔ kit-catalog: Kit accesses `kit-catalog.yaml` via SSH git URL (`git@github.com:nickpending/kit-catalog.git`).
- forge-assembler ↔ forge-armory: Assembler writes artifacts to `forge-armory/{type}/`, commits, and pushes. Kit registers skill/tool/agent/command artifacts only.
- forge-planner ↔ forge context: Planner reads/writes `~/.config/forge/context.json`.

Gotchas:
- Armory repo must be clean before assembler writes.
- `context.json` must NEVER be committed to git (use .gitignore).
- Level 3 execution testing (structural → reference → execution) must pass before armory commit.
- Automation configs (Justfiles, cron, n8n workflows) co-locate with their tool in `tools/{name}/`.
- Forge runtime XDG paths: `~/.config/forge/` (config), `~/.local/share/forge/` (state).
- Skill-scoped hooks (PreToolUse, Stop) live in the skill's `tools/` directory.
- Timing profiles (T1 cautious / T2 moderate / T3 aggressive) are reference constants for rate limiting.
- CONOPS documents live in `~/.local/share/forge/conops/`.
- Foundations loaded via `Skill("prompt-foundations")`, `Skill("skill-foundations")`, `Skill("command-foundations")`.
- Artifact templates located in `~/development/projects/forge/skills/forge-assembler/references/artifact-templates/`.
- Tool quality standards are defined in `tool-standards.md`.
