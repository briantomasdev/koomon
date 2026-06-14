# Koomon — Architect Mode Rules

Read [`CLAUDE.md`](../../CLAUDE.md) before any tool call. Rules here govern **how** Claude responds in Architect mode — enforcing the Spec-Driven Development (SDD) contract and ensuring specs are machine-actionable, not prose documents.

---

## 1. Architect Mode's Single Responsibility

Architect mode exists to produce **one output**: a finalized `specs/<feature-name>.md` that is ready to be handed to Code mode for the Red → Green → Refactor loop.

Do **not** write implementation code or test code in Architect mode. If asked, respond with the spec instead and note that Code mode executes the implementation.

---

## 2. Every Spec Must Contain All Eight Sections

Use [`specs/_template.md`](../../specs/_template.md) as the scaffold. A spec is not complete unless all eight sections are present and non-empty:

| # | Section | What makes it complete |
|---|---|---|
| 1 | Problem Statement | One paragraph; cites a user-visible symptom |
| 2 | Data Structures | Rust structs/enums + TypeScript payload types with exact field names (snake_case) |
| 3 | Interface / API Surface | Full function signatures with return types — no pseudocode |
| 4 | SQL Schema Changes | Migration filename + exact `CREATE TABLE` / `ALTER TABLE` DDL |
| 5 | Behaviour & Business Rules | Bullet list of verifiable rules, each checkable by a test |
| 6 | Edge Cases | Table mapping scenario → expected behaviour |
| 7 | Acceptance Criteria | Numbered `AC-NN` items, each mapping 1-to-1 to a future test case |
| 8 | Out of Scope | Explicit list preventing scope creep |

---

## 3. Acceptance Criteria Rules

Each `AC-NN` entry must be:
- **Verifiable** — a passing or failing assertion, not a vague goal.
- **Atomic** — one behaviour per criterion.
- **Unambiguous** — another engineer can write the test without asking clarifying questions.

❌ Bad: `AC-01: The sync works correctly.`
✅ Good: `AC-01: Applying two concurrent offline updates to the same CRDT Doc produces the same final state regardless of application order.`

---

## 4. Interface Definitions Must Be Exact

Rust function signatures in the spec become the **contract** that Code mode implements against. They must include:
- Full parameter list with types
- Return type (`Result<T, Error>`)
- `async` qualifier where appropriate
- The crate-level `Error` type (`crate::Error` or the domain-specific variant)

No hand-wavy pseudocode like `fn do_thing(input) -> output`.

---

## 5. Stack Constraints Apply to Specs

Specs must not propose architectures that violate `CLAUDE.md` constraints. Specifically:
- Data storage → `rusqlite` + SQLCipher. No alternative DB.
- Network → `wtransport` (QUIC). No WebSocket alternatives.
- CRDT → `yrs`. No `automerge-rs` without explicit roadmap approval.
- Async → `tokio`. No `async-std` or `smol`.
- UI → React + CSS Modules. No Tailwind, no inline styles.

If a proposed design would require a forbidden dependency, call it out in **Section 8 (Out of Scope)** and note the constraint.

---

## 6. Response Shape

- Output the spec file directly using `write_to_file` to `specs/<feature-name>.md`.
- When refining an existing spec, use `apply_diff` — do not reprint the whole file.
- No preamble. Begin directly with the file write.
- After writing, list only the **Open Questions** (Section 9) that remain unresolved and need user input before the spec can be marked Approved.
- Do not ask questions that can be answered by reading `CLAUDE.md` or `docs/ARCHITECTURE.md`.

---

## 7. Spec Lifecycle

```
Draft → (user reviews) → Approved → (Code mode begins Phase 2)
```

- A spec is **Draft** until all Open Questions are resolved.
- A spec is **Approved** when the user explicitly confirms it.
- Code mode must not start Phase 2 (failing tests) until the spec status is **Approved**.
- After implementation, update the spec's acceptance criteria checkboxes to reflect which are passing.

---

## 8. Always Skip

- Implementation code in spec files.
- Vague acceptance criteria ("it should work", "handles errors gracefully").
- Dependency install commands.
- Prose summaries of what was just written — end at the last diff or file write.
