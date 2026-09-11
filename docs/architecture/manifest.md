---
type: manifest
project: forge
generated: 2026-09-11
source: /Users/rudy/development/projects/forge/docs/architecture
reconciled_at: 26969b8e5c66a7b4ce8204176b8084d7bd93ace5
---

### Components

- **forge-strategist** — CONOPS from sensemaker decomposition; composition model (forked, opus).
- **forge-planner** — Reads approved CONOPS, produces a plan (forked).
- **forge-assembler** — Generates artifacts, 3-level verification, commits to armory.
- **forge skill** — Workflow router: init, campaign, improve.
- **forge-armory** — Repo storing ALL artifact types. → components/forge-armory.md
- **kit-catalog** — Registry backing store. → components/kit-catalog.md
- **forge runtime** — Shared XDG config/state.
- **CONOPS** — Strategist→planner interface doc.
- **composition model** — Strategist refs: shapes × substrates, taxonomy, evals, data deps.

### Where to look

Base dir: /Users/rudy/development/projects/forge/docs/architecture/

- Overview: architecture.md
- Components: components.md
- Decisions: decisions.md
- Contracts: boundaries.md
