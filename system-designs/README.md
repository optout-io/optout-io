# optout-io System Designs

Documentation for the optout-io platform architecture, decisions, and feature designs.

## Structure

| Path | Purpose |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Living current-state overview: component map, responsibilities, invariants |
| [decisions/](./decisions/) | Architecture Decision Records — immutable, numbered, explain *why* |
| [designs/](./designs/) | Feature design docs — requirements, cross-component impact, outcomes |

## How to use this

**Understanding the system:** Start with `ARCHITECTURE.md`.

**Understanding why something was built a certain way:** Browse `decisions/`.

**Working on a feature:** Create a doc in `designs/` before writing code. Record the requirements, which components are affected, what was decided, and (after shipping) what was actually implemented.

---

## Decisions

| # | Title | Status |
|---|---|---|
| [001](./decisions/001-orchestration-technology.md) | Orchestration Technology (Temporal, NATS, PostgreSQL) | Accepted |
| [002](./decisions/002-v1-system-design.md) | V1 System Design | Superseded |

## Designs

*(none yet — added per feature)*
