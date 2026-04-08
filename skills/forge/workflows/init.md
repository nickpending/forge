# /forge init

Initialize the forge runtime. Idempotent — safe to run multiple times.

```bash
mkdir -p ~/.config/forge
mkdir -p ~/.local/share/forge/conops
```

If `~/.config/forge/context.json` does not exist, create it from the context sample template:
```bash
test -f ~/.config/forge/context.json || echo "Creating forge context from defaults..."
```
Write the default context to `~/.config/forge/context.json`:

```json
{
  "operator": {
    "tools": [],
    "tools_missing": [],
    "infrastructure": {
      "resolver": null,
      "datasets": [],
      "data_dir": null
    },
    "preferences": {
      "workflow_style": "methodical",
      "reporting_format": "markdown"
    },
    "skill_level": "senior"
  },
  "campaigns": []
}
```

The strategist will populate this with real environment data on first campaign run.

Report:
```
Forge runtime initialized:
  Config: ~/.config/forge/
  Data: ~/.local/share/forge/
  Context: ~/.config/forge/context.json
```

STOP. Do not proceed to any other workflow.
