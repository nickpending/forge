# Kit Integration for Planning

Kit (`@voidwire/kit`) is the cross-cutting component registry. The planner queries Kit after artifact-type determination to find available components that match the plan's requirements.

---

## Kit CLI Commands

```bash
kit init <catalog-repo-url>     # Bootstrap
kit add --name X --repo Y --path Z --type skill --domain d --tags t  # Register
kit use <name>                  # Install to XDG path
kit use <name> --dir ./         # Project-scoped install
kit list --type skill --domain security  # Query
kit search <query>              # Keyword search
kit sync                        # Update all installed
kit push <name>                 # Push local changes to source repo
```

Install locations by type:
- `skill` -> `~/.claude/skills/<name>/`
- `command` -> `~/.claude/commands/<name>.md`
- `tool` -> `~/.local/bin/`
- `agent` -> `~/.config/sable/agents/`
- `harness` -> user workspace dir (e.g. `~/forge-harnesses/<name>/`); `kit use` clones + runs install

Kit catalog is a YAML file in its own git repo. State tracked at `~/.local/share/kit/state.yaml`. Config at `~/.config/kit/config.toml`.

---

## When to Query

The planner queries Kit **after artifact-type determination**, not before. The sequence is:

1. Understand intent
2. Determine artifact type (using decision tree from `forge-artifacts.md`)
3. Determine runtime
4. **Query Kit** for available components matching the plan's needs
5. Populate `components_available` in plan frontmatter
6. Identify gaps (components needed but not in Kit)
7. Classify each gap component: `reusability` (reusable / parent-scoped / private)
8. Pass to assembler (when `components_needed[]` has items to create)

Kit results inform what the assembler can reuse vs. what it must create. They do not influence artifact-type selection.

---

## Relevant Queries During Planning

| Query | When to use |
|-------|-------------|
| `kit list --type skill --domain security` | Plan includes skill components |
| `kit list --type tool --domain security` | Plan includes tool components |
| `kit list --type agent` | Plan includes agent components |
| `kit list --type command` | Plan includes command orchestrators |
| `kit list --type harness` | Plan includes harness components |
| `kit search <name>` | Check if a specific component exists |
| `kit search tag:bundle:<type>:<slug>` | Find all components bundled with a parent |

---

## Graceful Degradation

Kit not initialized: Planner degrades gracefully — `components_available` stays empty, plan notes Kit was unavailable. Assembler skips `kit add`/`kit use` but still commits artifacts to the armory.

The planner must work without Kit. Kit makes the assembler faster by reusing existing components, but the planner can produce valid plans with an empty component inventory. The assembler will create what is needed.

---

## What Kit Indexes

Five Kit-registered artifact types:

| Type | What Kit tracks | Examples |
|------|-----------------|---------|
| `skill` | Methodology skills — inline or forked, declarative knowledge | recon-methodology, idor-hunter, finding-verifier |
| `tool` | Deterministic tools with JSON I/O contracts or CLI interfaces | nuclei-template-search, httpx-parser |
| `agent` | Persona definitions with associated skill sets | recon-investigator, detection-builder |
| `command` | In-Claude-Code orchestrators coordinating agents via Skill() chain | forge, cve-response |
| `harness` | Agent SDK projects — standalone LLM-driven orchestrators | overnight-recon, n-day-response |

**Not Kit-indexed:** automation configs (armory-only, co-locate with parent tool).

Each entry has metadata for selection: artifact type, domain, tags (including `bundle:` tags for parent-scoped components), I/O contracts, dependencies, and validation status.
