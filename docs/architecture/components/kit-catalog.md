---
type: architecture
subtype: component-detail
project: "forge"
component: "kit-catalog"
status: active
created: "2026-03-24"
updated: "2026-03-24"
tags: [architecture, components]
---

# kit-catalog

**Purpose:** Kit component registry backing store — YAML catalog in its own git repo.

## Current State

Initialized with empty catalog placeholder. No artifacts registered yet.

- Repository: `git@github.com:nickpending/kit-catalog.git` (private)
- Local clone: `~/development/projects/kit-catalog/`
- Catalog file: `kit-catalog.yaml` (empty placeholder with `entries: []`)
- Kit cache: `~/.cache/kit/catalog/` (cloned on `kit init`)
- Kit config: `~/.config/kit/config.toml` (catalog repo URL)
- Kit state: `~/.local/share/kit/state.yaml` (lazy-created on first `kit use`)

## Design

Kit's catalog uses type-keyed sections (skills:, agents:, etc.) per the Kit README. The current empty placeholder uses a flat `entries: []` catch-all that will be superseded once artifacts are registered via `kit add`.

## Connections

- **Managed by:** Kit CLI (add, list, sync, search)
- **Populated by:** forge-assembler (registers artifacts after generation)
- **Consumed by:** forge-planner (queries existing catalog for reuse)
- **External dependency:** SSH access to GitHub required for kit list/sync operations
