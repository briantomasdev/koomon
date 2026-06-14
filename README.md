# Koomon

> **A local-first, cross-platform team messenger built with Rust and Tauri.**

Koomon is a commercial open-source (COSS) communication platform engineered for privacy, resilience, and zero-cloud sovereignty. Its unified Rust core runs identically on macOS, Windows, Linux, iOS, and Android — giving teams full ownership of their message history with no vendor lock-in.

---

## Table of Contents

- [Why Koomon](#why-koomon)
- [Architecture Overview](#architecture-overview)
- [Business Model](#business-model)
- [Development Roadmap](#development-roadmap)
- [Development Workflow](#development-workflow)
- [Getting Started](#getting-started)
- [Contributing](#contributing)
- [License](#license)

---

## Why Koomon

| Pain Point | Koomon's Answer |
|---|---|
| SaaS messengers own your data | All messages stored locally in SQLite; you own the file |
| No internet = no communication | Local-first CRDT engine syncs offline edits on reconnect |
| Mobile background sync is unreliable | Silent APNs/FCM push wakes the Rust core on demand |
| Self-hosting is operationally complex | Single `docker-compose up` server deployment |
| Enterprise AI search leaks data to the cloud | On-device vector RAG via SwarmState AI add-on |

---

## Architecture Overview

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full technical specification including:

- **Shared Rust Core** — Y-CRDT engine, SQLite storage, WebTransport (QUIC/HTTP3)
- **Desktop Adapter** — Tauri shell with persistent background daemon
- **Mobile Adapter** — Swift (iOS) and Kotlin (Android) plugins with silent push lifecycle

```
+-------------------------------------------------------+
|                  UI Presentation Layer                |
|                (React / TypeScript SPA)               |
+-------------------------------------------------------+
                           |
             Tauri IPC Bridge / FFI Bindings
                           |
+-------------------------------------------------------+
|               Cross-Platform Rust Core                |
|  +------------------+           +------------------+  |
|  |  Y-CRDT Engine   |           |  WebTransport    |  |
|  +------------------+           +------------------+  |
|  |  SQLite Storage  |           |  Background Sync |  |
|  +------------------+           +------------------+  |
+-------------------------------------------------------+
     /                      |                      \
[Desktop OS API]    [Apple APNs Binding]    [Google FCM Binding]
(Persistent Daemon) (30s Native Lifecycle)  (Transient Service)
```

---

## Business Model

See [`docs/BUSINESS_MODEL.md`](docs/BUSINESS_MODEL.md) for the full tier breakdown and MRR calculator model.

| Tier | License | Target | Price |
|---|---|---|---|
| **Community Core** | AGPLv3 | Families, privacy advocates, solo devs | Free |
| **Koomon Managed Cloud** | Proprietary SaaS | SMBs avoiding self-hosting ops | Per user/month |
| **SwarmState AI Enterprise** | Proprietary Add-on | Compliance-focused enterprise teams | Per user/month |

---

## Development Workflow

Koomon uses **Spec-Driven Development (SDD)**. The Markdown spec is the primary artifact — code is derived from it.

| File | Purpose |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Project constitution — tech stack, architectural constraints, quality rules. Read by all AI assistants automatically. |
| [`specs/`](specs/) | One `<feature-name>.md` per feature. Must be written and approved **before** any implementation or tests. |
| [`.roo/rules-code/token-efficiency.md`](.roo/rules-code/token-efficiency.md) | Code mode rules — enforces the Red → Green → Refactor loop and token-efficiency standards. |

### The Three-Phase Loop

```
1. Spec   →  Write specs/<feature>.md  (data structures, signatures, edge cases, acceptance criteria)
2. Red    →  Write failing tests that map to the spec
3. Green  →  Write minimum implementation to pass tests, then refactor
```

---

## Development Roadmap

See [`docs/ROADMAP.md`](docs/ROADMAP.md) for the full phase-by-phase execution plan.

| Phase | Focus | Timeline |
|---|---|---|
| 1 | Local-First Core (Rust workspace, Y-CRDT, SQLite) | Weeks 1–6 |
| 2 | Network Transport (wtransport, Tauri desktop MVP) | Weeks 7–12 |
| 3 | Native Mobile Integration (APNs, FCM, plugins) | Weeks 13–20 |
| 4 | Open-Source Launch (docs, Docker, public repo) | Weeks 21–24 |

---

## Getting Started

> ⚠️ Pre-release. The workspace is being initialized per the Phase 1 milestone.

### Prerequisites

- [Rust](https://rustup.rs/) ≥ 1.78
- [Node.js](https://nodejs.org/) ≥ 20 LTS
- [Tauri CLI](https://tauri.app/start/prerequisites/) v2
- Docker (for the self-hosted server)

### Self-Hosted Server (Community Core)

```bash
git clone https://github.com/your-org/koomon.git
cd koomon
docker compose up -d
```

### Desktop Development

```bash
cargo tauri dev
```

---

## Contributing

Koomon's Community Core is licensed under **AGPLv3**. Contributions to the core are welcome and must remain under AGPLv3. The `enterprise/` directory contains proprietary modules and is not open for external contribution.

Please read `CONTRIBUTING.md` before opening a pull request.

---

## License

- **Core** (`src/`, `server/`, `mobile/`): [GNU Affero General Public License v3.0](LICENSE)
- **Enterprise modules** (`enterprise/`): Proprietary — All rights reserved, Koomon Inc.
