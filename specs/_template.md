# Spec: [Feature Name]

> **Status:** Draft | Review | Approved  
> **Phase:** [Roadmap phase number]  
> **Author:** [name or handle]  
> **Last updated:** YYYY-MM-DD

---

## 1. Problem Statement

One paragraph. What user-visible problem does this feature solve? What breaks or is missing without it?

---

## 2. Data Structures

Define every struct, enum, and type this feature introduces or modifies. Use Rust syntax for core types; TypeScript for frontend payload types. These definitions become the authoritative contract.

```rust
// koomon-core/src/<module>/mod.rs

pub struct ExampleRecord {
    pub id: String,
    pub created_at: i64, // Unix timestamp (ms)
}

#[derive(Debug, thiserror::Error)]
pub enum ExampleError {
    #[error("record not found: {0}")]
    NotFound(String),
    #[error(transparent)]
    Db(#[from] rusqlite::Error),
}
```

```ts
// src/types/<domain>.types.ts

export type ExampleRecord = {
  id: string;
  created_at: number; // Unix timestamp (ms) — mirrors Rust field names exactly
};
```

---

## 3. Interface / API Surface

### Rust — Public Functions

List every `pub fn` / `pub async fn` this feature exposes. Include full signature and a one-line description.

```rust
/// Inserts a new record and returns the assigned id.
pub async fn create_example(
    conn: &rusqlite::Connection,
    payload: CreateExamplePayload,
) -> Result<String, ExampleError>;

/// Fetches all records for the given owner.
pub async fn list_examples(
    conn: &rusqlite::Connection,
    owner_id: &str,
) -> Result<Vec<ExampleRecord>, ExampleError>;
```

### Tauri Commands (if applicable)

```rust
// src-tauri/src/commands/<domain>.rs
#[tauri::command]
pub async fn create_example_cmd(payload: CreateExamplePayload) -> Result<String, String>;
```

### TypeScript API Wrappers (if applicable)

```ts
// src/api/<domain>.ts
export const createExample: (payload: CreateExamplePayload) => Promise<string>;
export const listExamples: (ownerId: string) => Promise<ExampleRecord[]>;
```

---

## 4. SQL Schema Changes

List every new table or column. Include the migration file name.

**Migration:** `db/migrations/<NNN>_<desc>.sql`

```sql
CREATE TABLE IF NOT EXISTS example_records (
    id          TEXT PRIMARY KEY,
    owner_id    TEXT NOT NULL,
    created_at  INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_example_records_owner ON example_records (owner_id);
```

---

## 5. Behaviour & Business Rules

Bullet list. Each bullet is one verifiable rule.

- [ ] A record with a duplicate `id` must return `ExampleError::AlreadyExists`.
- [ ] `list_examples` returns records sorted by `created_at` ascending.
- [ ] Deleting an owner cascades to all their records.

---

## 6. Edge Cases

| Scenario | Expected behaviour |
|---|---|
| DB file is locked by another process | Return `ExampleError::Db(rusqlite::Error::SqliteFailure(...))` — do not panic |
| `owner_id` is an empty string | Return `ExampleError::InvalidInput("owner_id cannot be empty")` |
| Network is offline during creation | Record is created locally; `pending_sync` entry is inserted atomically |

---

## 7. Acceptance Criteria

Each criterion maps 1-to-1 to a test case in Phase 2 (Red). A criterion is "done" when its test is green.

- [ ] `AC-01`: `create_example` with valid payload inserts a row and returns a non-empty `id`.
- [ ] `AC-02`: `create_example` with a duplicate `id` returns `Err(ExampleError::AlreadyExists)`.
- [ ] `AC-03`: `list_examples` for an owner with 0 records returns `Ok(vec![])`.
- [ ] `AC-04`: `list_examples` returns results sorted by `created_at` ascending.
- [ ] `AC-05`: Creating a record while offline inserts a corresponding row in `pending_sync`.

---

## 8. Out of Scope

Explicitly list what this spec does **not** cover to prevent scope creep.

- Pagination of `list_examples` results (deferred to a follow-up spec).
- UI components for displaying records (covered in `specs/example-ui.md`).

---

## 9. Open Questions

Track unresolved design decisions here. Resolve before marking status → Approved.

| # | Question | Owner | Resolution |
|---|---|---|---|
| 1 | Should `id` be a UUID v4 or a ULID? | — | — |
