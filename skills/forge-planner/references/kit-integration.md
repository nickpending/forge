# Kit Integration for Planning

Kit (`@voidwire/kit`) is the cross-cutting component registry. The planner queries Kit after tier determination to find available components that match the plan's requirements.

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
- `script` -> `~/.local/bin/`
- `agent` -> `~/.config/sable/agents/`

Kit catalog is a YAML file in its own git repo. State tracked at `~/.local/share/kit/state.yaml`. Config at `~/.config/kit/config.toml`.

---

## When to Query

The planner queries Kit **after tier determination**, not before. The sequence is:

1. Understand intent
2. Determine tier (using rubric from forge-tiers.md)
3. Select artifact pattern
4. **Query Kit** for available components matching the plan's needs
5. Populate `components_available` in plan frontmatter
6. Identify gaps (components needed but not in Kit)
7. Pass to assembler (Tier 3+)

Kit results inform what the assembler can reuse vs. what it must create. They do not influence tier selection.

---

## Relevant Queries During Planning

| Query | When to use |
|-------|-------------|
| `kit list --type skill --domain security` | Find existing methodology skills for Tier 2+ plans |
| `kit list --type wrapper --domain security` | Find existing wrappers for Tier 3+ plans |
| `kit search <tool-name>` | Check if a wrapper exists for a required tool |
| `kit list --type agent --domain security` | Find existing agent personas for Tier 4+ plans |

---

## Graceful Degradation

Kit not initialized: Planner degrades gracefully — `components_available` stays empty, plan notes Kit was unavailable. Assembler emits kit-manifest.yaml but skips `kit add`/`kit use`.

The planner must work without Kit. Kit makes the assembler faster by reusing existing components, but the planner can produce valid plans with an empty component inventory. The assembler will create what is needed.

---

## What Kit Indexes

- **Pattern 0 skills** — methodology skills (recon, triage, detection), declarative knowledge
- **Pattern 1 forked skill-agents** — self-contained autonomous tasks
- **Pattern 2 agent personas** — identity definitions with associated skill sets
- **Wrapper scripts** — deterministic tools with JSON I/O contracts
- **Tool metadata** — what tools are available, what they do, how to use them

Each entry has metadata for selection: pattern type, domain, tier support, I/O contracts, dependencies, and validation status.
