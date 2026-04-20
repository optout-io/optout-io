# optout-io

Privacy opt-out automation platform. Automates GDPR/CCPA/DPPA erasure requests across hundreds of data brokers on behalf of users.

## Workspace

This repo is the system-level context for the optout-io platform. Component services live in their own repos.

| Repo | Role |
|---|---|
| [data-erasure-wf](https://github.com/optout-io/data-erasure-wf) | Core Temporal orchestration service for data erasure workflows |
| [lake](https://github.com/optout-io/lake) | Event ingestion and query service (NATS → PostgreSQL) |
| [abscond](https://github.com/optout-io/abscond) | Internal admin dashboard with real-time workflow timelines |
| [webform-playwright](https://github.com/optout-io/webform-playwright) | Browser automation workers for 600+ data broker opt-out forms |
| [contracts](https://github.com/optout-io/contracts) | Shared protobuf contract definitions (Go + TypeScript) |
| [go-common](https://github.com/optout-io/go-common) | Shared Go infrastructure library |
| [adr](https://github.com/optout-io/adr) | Legacy architecture decision records (superseded by system-designs/) |

## Documentation

- **[system-designs/ARCHITECTURE.md](./system-designs/ARCHITECTURE.md)** — current system overview: component map, responsibilities, invariants
- **[system-designs/decisions/](./system-designs/decisions/)** — architecture decision records
- **[system-designs/designs/](./system-designs/designs/)** — per-feature design docs
