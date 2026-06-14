# Koomon — Code Mode Rules

Read [`CLAUDE.md`](../../CLAUDE.md) before any tool call. Rules here govern **how** Claude responds in Code mode — enforcing the SDD loop, mandatory crates, and token efficiency.

---

## 1. Spec-Driven Development Loop (Mandatory Order)

### Phase 1 — Spec
If no `specs/<feature>.md` exists:
- Write it. Define: data structures, function/command signatures, edge cases, acceptance criteria.
- **Stop. Do not write code or tests until the user confirms the spec.**

### Phase 2 — Red (failing tests)
- Write tests that map 1-to-1 to the spec's acceptance criteria.
- Tests must fail at this point — zero implementation.
- Rust: `tests/<module>_test.rs` or `#[cfg(test)]` block. TypeScript: `*.test.ts` / `*.test.tsx`.

### Phase 3 — Green → Refactor
- Write the **minimum** code to pass the tests.
- Refactor for performance and clarity. Tests must stay green.
- Rust: reduce allocations, prefer `tokio::select!`. TypeScript: extract hooks, remove inline styles.

---

## 2. Enforced Crates & Packages (Do Not Substitute)

These are the **only** approved libraries for each concern. Never introduce an alternative without an explicit user decision.

### Rust — Mandatory Crates

| Concern | Crate | Min Version | Notes |
|---|---|---|---|
| CRDT engine | `yrs` | ≥ 0.21 | One `Doc` per thread. `automerge-rs` only if roadmap approves. |
| Local storage | `rusqlite` | ≥ 0.31 | Must use `features = ["bundled", "sqlcipher"]`. |
| Network / QUIC | `wtransport` | ≥ 0.5 | One `Session` per process; channels as `Stream`s. |
| Async runtime | `tokio` | ≥ 1.37 | `features = ["full"]`. No `async-std`, no `smol`. |
| Serialization | `serde` + `serde_json` | ≥ 1.0 | Derive `Serialize`/`Deserialize` — never manual impls. |
| Library errors | `thiserror` | ≥ 1.0 | One `Error` enum per crate. |
| Binary errors | `anyhow` | ≥ 1.0 | Only in `main.rs` and Tauri command handlers. |
| App shell / IPC | `tauri` | v2 | No Electron. No alternative shell. |
| Crypto (E2E) | `sodiumoxide` or `libsodium-sys` | latest | NaCl box (X25519 + XSalsa20-Poly1305) for CRDT envelopes. |

**Forbidden substitutions:**
- ❌ `async-std`, `smol` → use `tokio`
- ❌ `diesel`, `sea-orm`, `sqlx` → use `rusqlite`
- ❌ `tungstenite`, `tokio-tungstenite` (WebSocket) → use `wtransport`
- ❌ `automerge-rs` without roadmap approval → use `yrs`
- ❌ `ring`, `rustls` (direct) for message crypto → use `sodiumoxide`/`libsodium-sys`

### TypeScript / Node — Mandatory Packages

| Concern | Package | Notes |
|---|---|---|
| UI framework | `react` ≥ 18 | No Vue, Svelte, or solid-js. |
| Type system | `typescript` ≥ 5.4 | `strict: true` always. |
| Tauri JS bridge | `@tauri-apps/api` v2 | All `invoke` / `listen` calls go through typed wrappers in `src/api/`. |
| Styling | CSS Modules (`*.module.css`) | No Tailwind, no CSS-in-JS, no inline `style={{}}`. |
| Test runner | Vitest | No Jest unless project already uses it. |

**Forbidden substitutions:**
- ❌ Axios, `node-fetch` → data comes from Tauri `invoke`, not HTTP in the renderer
- ❌ Redux, Zustand, Jotai → use React Context unless roadmap approves
- ❌ styled-components, emotion → use CSS Modules

---

## 3. Response Shape

- Output **only what changes**. Never reprint unchanged code.
- `apply_diff` for existing files. `write_to_file` for new files only.
- No preamble. Begin directly with the tool call or code block.
- One sentence of rationale per change — only when non-obvious.
- No closing summary. End at the last code or tool output.

---

## 4. File Separation — Always Enforced

| Concern | File pattern |
|---|---|
| Implementation | `*.rs` / `*.ts` / `*.tsx` |
| Tests | `tests/*.rs` · `*.test.ts` · `*.test.tsx` — **always a separate file** |
| Styles | `*.module.css` — never `style={{}}` inline |
| TypeScript types | `types/<domain>.types.ts` |
| SQL migrations | `db/migrations/<NNN>_<desc>.sql` |
| Tauri command handlers | `commands/<domain>.rs` |
| Tauri API wrappers | `api/<domain>.ts` |
| Shared React hooks | `hooks/use<Name>.ts` |

**No monolithic files.** Split at ~150 lines by responsibility before adding more code.

---

## 5. Human-Maintainability Rules

- One-line doc comment on every exported function/command (`///` Rust, `/** */` TypeScript).
- Descriptive names. Single-letter variables only for loop iterators (`i`, `j`).
- Magic numbers and string literals → named constants at the top of the file.
- Max 3 levels of indentation. Extract to a named helper beyond that.
- `// --- Section Name ---` headers in longer files.
- No glob imports (`use crate::*`, `import * from`).

---

## 6. Rust Core (`koomon-core`)

### Async
- `tokio` only. No `std::thread` for I/O.
- `tokio::select!` over nested `.await` chains.
- Tests: `#[tokio::test]`.

### Errors
- `thiserror` Error enum per crate. `anyhow` only in `main.rs` / command handlers.
- `Result<T, crate::Error>` everywhere. No `.unwrap()` / `.expect()` outside tests.
- Third-party errors mapped with `#[from]`.

### SQLite (`rusqlite`)
- Features: `["bundled", "sqlcipher"]` — both required, no exceptions.
- Migrations: `db/migrations/<NNN>_<desc>.sql`, applied in version order at startup.
- Positional bind params (`?1`, `?2`). No SQL string interpolation.
- Multi-statement operations → `transaction`.
- Row lists: `query_map` + `collect::<Result<Vec<_>>>()`.

### Y-CRDT (`yrs`)
- One `Doc` per conversation thread.
- `transact_mut()` for writes, `transact()` for reads.
- NaCl-box the `Update` blob → persist to `pending_sync` → then acknowledge.

### WebTransport (`wtransport`)
- One `Session` per process; channels as bidirectional `Stream`s within it.
- Reconnect: `min(2^attempt * 200ms, 30s)`.
- Send paths cancel-safe inside `tokio::select!`.

### Rust Tests
- In `tests/<module>_test.rs` or `#[cfg(test)]` block — never above production logic.
- Naming: `test_<unit>_<scenario>`.
- No `println!`. Use `assert_eq!` / `assert_matches!`.

---

## 7. Tauri IPC

- Every `#[tauri::command]` is `async`, returns `Result<T, String>`.
- Error → `String` conversion at the command boundary only.
- Never clone `AppHandle` into long-lived tasks.
- Server-push via `app.emit(...)`. No polling from TypeScript.
- `main.rs` = builder setup only. Handlers in `commands/<domain>.rs`.

---

## 8. React / TypeScript

### Tauri Wrappers (`src/api/<domain>.ts`)
```ts
export const sendMessage = (channelId: string, content: string) =>
  invoke<void>("send_message", { channelId, content });
```
- `listen` in `useEffect`; return `UnlistenFn` as cleanup.
- Never call `@tauri-apps/api` directly inside a component.

### State
- Local: `useState` / `useReducer`.
- Shared: React Context.
- Server data: fetched once on mount, updated via `listen` only. No timers.

### TypeScript Standards
- Payload types use snake_case to mirror Rust struct fields exactly.
- `strict: true`. No `any`. No `@ts-ignore` without same-line justification.
- `type` for data shapes; `interface` for extensible contracts.

### TypeScript Tests (Vitest)
- Always a separate `*.test.tsx` / `*.test.ts` file.
- Mock `invoke` at the `src/api/` boundary, not inside components.
- Mirror source path: `src/api/messaging.ts` → `src/api/messaging.test.ts`.

---

## 9. Security (Non-Negotiable)

- Never log plaintext content, keys, JWT tokens, or device identity keys.
- SQLCipher key from OS credential store only (Keychain / DPAPI / Secret Service / Android Keystore).
- CRDT blobs NaCl-boxed via `sodiumoxide`/`libsodium-sys` before every `pending_sync` `INSERT`.

---

## 10. Always Skip

- `println!` / `dbg!` in production code.
- `TODO` / `FIXME` placeholder comments.
- `#[allow(dead_code)]` / `#[allow(unused)]`.
- Dependency install commands (`cargo add`, `npm install`).
- Full file reprints for single-function changes.
- Boilerplate `impl Display` when `thiserror` covers it.
