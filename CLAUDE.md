# CLAUDE.md — Koomon Project Constitution

This file is the **single source of truth** for every AI assistant working in this repository. Read it before touching any file. Rules here override any default behavior.

---

## 1. What Koomon Is

Koomon is a **local-first, cross-platform team messenger** built with Rust and Tauri. It targets macOS, Windows, Linux, iOS, and Android from a single Rust core.

**Non-negotiable constraints:**

| Constraint | Rule |
|---|---|
| Local-first | Every read/write hits local SQLite first. The network is a sync mechanism, never a requirement. |
| Privacy by default | End-to-end encryption is a property of the CRDT envelope. The server is a relay, never an authority. |
| Single-engineer maintainable | UI and sync logic are strictly decoupled. One Rust engineer must be able to maintain core parity across all platforms. |

---

## 2. Tech Stack (Do Not Deviate)

| Layer | Technology |
|---|---|
| Shared core | Rust (`koomon-core` Cargo workspace library) |
| CRDT engine | `yrs` ≥ 0.21 |
| Local storage | `rusqlite` ≥ 0.31 (bundled) + SQLCipher |
| Network | `wtransport` ≥ 0.5 (WebTransport / QUIC / HTTP3) |
| Async runtime | `tokio` ≥ 1.37 |
| App shell | Tauri v2 |
| UI | React ≥ 18 + TypeScript ≥ 5.4 (`strict: true`) |
| Error handling | `thiserror` in libraries, `anyhow` only in binaries |

**Never introduce:** cloud databases, polling timers for server data, Electron, Redux/Zustand (unless roadmap explicitly adds it), raw `invoke` calls in React components.

---

## 3. Canonical Directory Layout

```
CLAUDE.md                        ← this file
specs/                           ← functional specs (source of truth for features)
koomon-core/src/
  lib.rs                         ← pub mod declarations only
  error.rs                       ← crate Error enum (thiserror)
  db/
    mod.rs
    migrations/                  ← 001_init.sql, 002_*.sql …
    queries/                     ← typed query helpers per table
  crdt/mod.rs
  transport/mod.rs
  sync/mod.rs
  tests/                         ← integration tests
src-tauri/src/
  main.rs                        ← Tauri builder setup only
  commands/                      ← one file per domain
src/ (frontend)
  api/                           ← typed invoke wrappers (never raw invoke in components)
  components/<Name>/
    <Name>.tsx
    <Name>.module.css
    <Name>.test.tsx
  hooks/                         ← use<Name>.ts
  types/                         ← <domain>.types.ts
```

---

## 4. Spec-Driven Development Workflow (SDD)

**The spec is the primary artifact. Code is derived from the spec.**

### Step 1 — Write the Spec First
Before any implementation, a `specs/<feature-name>.md` file must exist and be finalized. It defines:
- Data structures and Rust/TypeScript interfaces
- Exact function/command signatures
- Edge cases and failure modes
- Acceptance criteria (verifiable, not vague)

### Step 2 — Write Failing Tests (Red)
With the spec approved, write tests that verify the acceptance criteria. **No implementation yet.** Tests must fail at this point.

### Step 3 — Implement to Pass Tests (Green → Refactor)
Write the minimum code to make the tests pass. Then refactor for performance and clarity while keeping tests green.

**Claude must follow this order when asked to implement a feature. If no spec exists, write the spec first and stop — do not proceed to code until the spec is confirmed.**

---

## 5. Code Quality Rules

### Rust
- `Result<T, crate::Error>` everywhere. No `.unwrap()` / `.expect()` outside tests.
- No monolithic files. Max ~150 lines per file; split by responsibility.
- No `println!` / `dbg!` in production code.
- No `#[allow(dead_code)]` / `#[allow(unused)]`.
- Tests in `tests/` directory or `#[cfg(test)]` block — never mixed into production logic.
- Test naming: `test_<unit>_<scenario>`.

### TypeScript / React
- `strict: true`. No `any`. No `@ts-ignore` without a same-line justification.
- Styles in `*.module.css` — never inline `style={{}}`.
- Tests always in a separate `*.test.tsx` / `*.test.ts` file.
- Payload types use snake_case to mirror Rust struct fields exactly.

### General
- One concern per file (logic, styles, tests, types, migrations — all separate).
- Max 3 levels of indentation; extract to named helper beyond that.
- Every exported function/command has a one-line doc comment.
- No magic numbers or string literals — use named constants.

---

## 6. Security Rules (Non-Negotiable)

- Never log plaintext message content, encryption keys, JWT tokens, or device identity keys.
- SQLCipher key must come from the OS credential store (Keychain / DPAPI / Secret Service / Android Keystore). Never from a raw user password.
- All CRDT blobs must be NaCl-boxed **before** writing to `pending_sync`.
- The server must never receive plaintext — only encrypted CRDT update blobs.

---

## 7. What Claude Should Skip (Unless Explicitly Asked)

- Dependency install commands (`cargo add`, `npm install`) — assume deps are in manifests.
- Full file reprints when only one function changes — use targeted diffs.
- Closing summaries after completing a task.
- Placeholder `TODO` / `FIXME` comments.
- Boilerplate `impl Display` when `thiserror` `#[error("…")]` covers it.
