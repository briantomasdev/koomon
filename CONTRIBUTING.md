# Contributing to Koomon

Koomon's Community Core is licensed under **AGPLv3**. All contributions to `src/`, `server/`, and `mobile/` must remain under AGPLv3. The `enterprise/` directory is proprietary and not open for external contribution.

---

## Before You Start

Read [`CLAUDE.md`](CLAUDE.md). It is the project constitution — tech stack, architectural constraints, and code quality rules. Every AI assistant and every human contributor is expected to follow it.

---

## Workflow: Spec-Driven Development (SDD)

Koomon uses a three-phase loop. **Do not open a PR without going through all three phases.**

### Phase 1 — Write the Spec

Before writing any code or tests, create `specs/<feature-name>.md` using [`specs/_template.md`](specs/_template.md) as the scaffold.

A spec must define:
- Data structures (Rust structs/enums + TypeScript payload types)
- Function and command signatures with exact return types
- SQL schema changes with migration file name
- Business rules (each verifiable by a test)
- Edge cases (scenario → expected behaviour)
- Numbered acceptance criteria (`AC-NN`) that map 1-to-1 to future tests

Open a PR or GitHub Discussion with the spec. **Get it approved before writing code.** This is the definition of done for Phase 1.

### Phase 2 — Red (Failing Tests)

With an approved spec, write the tests for every `AC-NN` acceptance criterion. Tests must fail at this point — no implementation yet.

- Rust: `tests/<module>_test.rs` or `#[cfg(test)]` block
- TypeScript: `*.test.ts` / `*.test.tsx` (always a separate file)
- Test naming: `test_<unit>_<scenario>`

### Phase 3 — Green → Refactor

Write the **minimum** implementation to make the tests pass. Then refactor for performance and clarity while keeping tests green.

---

## Code Standards (Summary)

Full rules are in [`.roo/rules-code/token-efficiency.md`](.roo/rules-code/token-efficiency.md). Key points:

| Rule | Detail |
|---|---|
| One concern per file | Logic, tests, styles, types, migrations — always separate files |
| No monolithic files | Split at ~150 lines by responsibility |
| Mandatory crates only | `yrs`, `rusqlite` (bundled + sqlcipher), `wtransport`, `tokio`, `thiserror`/`anyhow`, `tauri` v2, `sodiumoxide` — see `CLAUDE.md` for the full list |
| No forbidden substitutions | No `async-std`, `diesel`, `tungstenite`, Redux, Tailwind, etc. |
| Exported functions need doc comments | `///` in Rust, `/** */` in TypeScript |
| Max 3 indentation levels | Extract to a named helper beyond that |
| No inline styles in React | Use `*.module.css` — never `style={{}}` |

---

## Security Rules

These are non-negotiable. PRs that violate them will not be merged.

- Never log plaintext message content, encryption keys, JWT tokens, or device identity keys.
- SQLCipher key must come from the OS credential store — never from a raw user password.
- All CRDT blobs must be NaCl-boxed **before** being written to `pending_sync`.

---

## Pull Request Checklist

Before opening a PR, confirm:

- [ ] A `specs/<feature-name>.md` exists and is Approved
- [ ] Tests are in separate files and named `test_<unit>_<scenario>`
- [ ] All acceptance criteria (`AC-NN`) in the spec have a corresponding passing test
- [ ] No new dependencies outside the approved crate/package list
- [ ] No `println!` / `dbg!` in production code
- [ ] No `TODO` / `FIXME` comments left in changed files
- [ ] Exported functions have doc comments
- [ ] SQL changes have a versioned migration file in `db/migrations/`

---

## Getting Help

- Architecture questions → [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)
- Roadmap context → [`docs/ROADMAP.md`](docs/ROADMAP.md)
- Spec format → [`specs/_template.md`](specs/_template.md)
- AI assistant rules → [`CLAUDE.md`](CLAUDE.md)
