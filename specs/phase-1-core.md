# Spec: Phase 1 — Local-First Rust Core

> **Status:** Draft  
> **Phase:** 1 (Weeks 1–6)  
> **Author:** Koomon Core Team  
> **Last updated:** 2026-06-14

---

## 1. Problem Statement

Koomon requires a platform-agnostic Rust library (`koomon-core`) that stores messages locally in an encrypted SQLite database and represents each conversation thread as a Y-CRDT document. Without this core, no platform adapter (desktop or mobile) can function offline, and there is no convergent data model to build network sync on top of. Phase 1 delivers the storage and CRDT layers in isolation — no network, no UI — so they can be verified independently before transport is introduced in Phase 2.

---

## 2. Data Structures

```rust
// koomon-core/src/crdt/mod.rs

/// A handle to a single conversation thread's CRDT document.
pub struct ThreadDoc {
    pub channel_id: String,
    doc: yrs::Doc,
}

/// A serialized CRDT update blob ready for persistence or transmission.
pub struct EncodedUpdate {
    pub channel_id: String,
    /// Raw bytes from `TransactionMut::encode_update_v1()`.
    pub payload: Vec<u8>,
}
```

```rust
// koomon-core/src/db/mod.rs

/// A single appended message record stored in `messages`.
pub struct MessageRecord {
    pub id: String,
    pub channel_id: String,
    pub author_id: String,
    /// Raw CRDT update blob (encrypted before insert; decrypted after fetch).
    pub crdt_update: Vec<u8>,
    /// Lamport-style vector clock serialized as JSON.
    pub vector_clock: String,
    pub created_at: i64, // Unix timestamp (ms)
}

/// A channel metadata record.
pub struct ChannelRecord {
    pub id: String,
    pub name: String,
    pub workspace_id: String,
    pub channel_type: String, // "text" | "voice"
}

/// An outbound CRDT update not yet acknowledged by the relay server.
pub struct PendingSyncRecord {
    pub id: String,       // UUID v4
    pub channel_id: String,
    /// NaCl-boxed CRDT update blob.
    pub encrypted_payload: Vec<u8>,
    pub created_at: i64,
}
```

```rust
// koomon-core/src/error.rs

#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("database error: {0}")]
    Db(#[from] rusqlite::Error),
    #[error("record not found: {0}")]
    NotFound(String),
    #[error("invalid input: {0}")]
    InvalidInput(String),
    #[error("crdt encode error: {0}")]
    CrdtEncode(String),
}
```

```ts
// src/types/core.types.ts

export type MessageRecord = {
  id: string;
  channel_id: string;
  author_id: string;
  crdt_update: number[]; // byte array
  vector_clock: string;
  created_at: number;
};

export type ChannelRecord = {
  id: string;
  name: string;
  workspace_id: string;
  channel_type: string;
};

export type PendingSyncRecord = {
  id: string;
  channel_id: string;
  encrypted_payload: number[];
  created_at: number;
};
```

---

## 3. Interface / API Surface

### Rust — Database Layer (`koomon-core/src/db/`)

```rust
/// Opens (or creates) the encrypted SQLite database at `path`, applies all
/// pending migrations, and returns a connection.
pub fn open_db(path: &std::path::Path, key: &str) -> Result<rusqlite::Connection, Error>;

/// Inserts a message record. The `crdt_update` field must already be NaCl-boxed.
pub fn insert_message(
    conn: &rusqlite::Connection,
    record: &MessageRecord,
) -> Result<(), Error>;

/// Returns all messages for `channel_id`, ordered by `created_at` ascending.
pub fn list_messages(
    conn: &rusqlite::Connection,
    channel_id: &str,
) -> Result<Vec<MessageRecord>, Error>;

/// Inserts a channel record. Returns `Error::InvalidInput` if `id` is empty.
pub fn insert_channel(
    conn: &rusqlite::Connection,
    record: &ChannelRecord,
) -> Result<(), Error>;

/// Fetches a channel by `id`. Returns `Error::NotFound` if absent.
pub fn get_channel(
    conn: &rusqlite::Connection,
    channel_id: &str,
) -> Result<ChannelRecord, Error>;

/// Inserts a pending-sync record atomically with the originating message insert.
/// Must be called inside a `transaction`.
pub fn insert_pending_sync(
    conn: &rusqlite::Connection,
    record: &PendingSyncRecord,
) -> Result<(), Error>;

/// Removes a pending-sync record once the relay has acknowledged it.
pub fn delete_pending_sync(
    conn: &rusqlite::Connection,
    pending_sync_id: &str,
) -> Result<(), Error>;

/// Returns all unacknowledged pending-sync records, ordered by `created_at` ascending.
pub fn list_pending_sync(
    conn: &rusqlite::Connection,
) -> Result<Vec<PendingSyncRecord>, Error>;
```

### Rust — CRDT Layer (`koomon-core/src/crdt/`)

```rust
/// Creates a new in-memory Y-CRDT document for `channel_id`.
pub fn new_thread_doc(channel_id: &str) -> ThreadDoc;

/// Applies a remote `Update` blob to `doc`. Idempotent for duplicate updates.
pub fn apply_update(doc: &ThreadDoc, update: &[u8]) -> Result<(), Error>;

/// Appends `content` to the thread's CRDT text, encodes the resulting update,
/// and returns the raw update blob for persistence.
pub fn append_text(
    doc: &mut ThreadDoc,
    author_id: &str,
    content: &str,
) -> Result<EncodedUpdate, Error>;

/// Returns the current full-document state vector (for sync negotiation).
pub fn state_vector(doc: &ThreadDoc) -> Vec<u8>;
```

---

## 4. SQL Schema Changes

**Migration:** `db/migrations/001_init.sql`

```sql
CREATE TABLE IF NOT EXISTS messages (
    id            TEXT PRIMARY KEY,
    channel_id    TEXT NOT NULL,
    author_id     TEXT NOT NULL,
    crdt_update   BLOB NOT NULL,
    vector_clock  TEXT NOT NULL,
    created_at    INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_messages_channel ON messages (channel_id, created_at);

CREATE TABLE IF NOT EXISTS channels (
    id            TEXT PRIMARY KEY,
    name          TEXT NOT NULL,
    workspace_id  TEXT NOT NULL,
    channel_type  TEXT NOT NULL DEFAULT 'text'
);

CREATE TABLE IF NOT EXISTS peers (
    id              TEXT PRIMARY KEY,
    workspace_id    TEXT NOT NULL,
    last_vector_clock TEXT NOT NULL DEFAULT '{}'
);

CREATE TABLE IF NOT EXISTS pending_sync (
    id                TEXT PRIMARY KEY,
    channel_id        TEXT NOT NULL,
    encrypted_payload BLOB NOT NULL,
    created_at        INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_pending_sync_created ON pending_sync (created_at);
```

---

## 5. Behaviour & Business Rules

- [ ] `open_db` must run all migrations in filename-version order on every startup; already-applied migrations must be skipped without error.
- [ ] `insert_message` and `insert_pending_sync` for the same logical operation must execute inside a single `transaction`; if either fails, both are rolled back.
- [ ] `list_messages` always returns results sorted by `created_at` ascending.
- [ ] `insert_channel` with an empty `id` must return `Error::InvalidInput`.
- [ ] `append_text` must use `doc.transact_mut()` and return the encoded `Update` blob; it must not write to the database directly.
- [ ] `apply_update` must use `doc.transact_mut()` and must be idempotent: applying the same update bytes twice must produce the same document state as applying once.
- [ ] `state_vector` must use `doc.transact()` (read-only) — never `transact_mut`.
- [ ] The SQLCipher encryption key passed to `open_db` must never be logged, printed, or included in any error message.

---

## 6. Edge Cases

| Scenario | Expected behaviour |
|---|---|
| Migration file is missing from the binary | `open_db` returns `Error::InvalidInput("migration N not found")` — do not silently skip |
| `insert_message` called with duplicate `id` | `rusqlite` unique constraint violation → `Error::Db(...)` — do not panic |
| `apply_update` receives a zero-length byte slice | Returns `Error::CrdtEncode("empty update payload")` |
| `list_messages` for a channel with no messages | Returns `Ok(vec![])` — not an error |
| `delete_pending_sync` with non-existent `id` | Returns `Ok(())` — idempotent, not an error |
| DB file is locked by another process at `open_db` | Returns `Error::Db(rusqlite::Error::SqliteFailure(...))` — caller decides retry strategy |
| Two `ThreadDoc`s for the same `channel_id` | Each is an independent in-memory doc; merging is the caller's responsibility via `apply_update` |

---

## 7. Acceptance Criteria

- [ ] `AC-01`: `open_db` creates the database file if it does not exist and applies `001_init.sql` without error.
- [ ] `AC-02`: `open_db` called a second time on an existing database does not re-apply migrations or return an error.
- [ ] `AC-03`: `insert_message` followed by `list_messages` on the same `channel_id` returns a `Vec` containing exactly that record.
- [ ] `AC-04`: `list_messages` returns records ordered by `created_at` ascending when multiple records exist.
- [ ] `AC-05`: `insert_message` with a duplicate `id` returns `Err(Error::Db(...))`.
- [ ] `AC-06`: `insert_channel` with an empty `id` returns `Err(Error::InvalidInput(...))`.
- [ ] `AC-07`: `get_channel` for a non-existent `id` returns `Err(Error::NotFound(...))`.
- [ ] `AC-08`: `insert_message` and `insert_pending_sync` executed in a transaction — if `insert_pending_sync` fails, `messages` is also rolled back (0 rows inserted).
- [ ] `AC-09`: `list_pending_sync` returns only records not yet deleted, ordered by `created_at` ascending.
- [ ] `AC-10`: `delete_pending_sync` with a non-existent `id` returns `Ok(())`.
- [ ] `AC-11`: `append_text` returns an `EncodedUpdate` with a non-empty `payload`.
- [ ] `AC-12`: `apply_update` called twice with identical bytes produces the same `state_vector` as calling it once.
- [ ] `AC-13`: `apply_update` with a zero-length slice returns `Err(Error::CrdtEncode(...))`.
- [ ] `AC-14`: Two `ThreadDoc`s with independent mutations, each applied to the other via `apply_update`, converge to the same `state_vector`.

---

## 8. Out of Scope

- Network transport (`wtransport` session, relay connection) — covered in `specs/phase-2-transport.md`.
- NaCl encryption of `crdt_update` blobs (the encryption wrapper is a Phase 2 concern; Phase 1 tests use plaintext blobs).
- Tauri IPC command handlers — covered in `specs/phase-2-transport.md`.
- React/TypeScript frontend components — covered in `specs/phase-2-transport.md`.
- `peers` table query helpers (schema is created in `001_init.sql` but no query functions are needed until Phase 2).

---

## 9. Open Questions

| # | Question | Owner | Resolution |
|---|---|---|---|
| 1 | Should `id` fields use UUID v4 or ULID? ULID gives time-sortable IDs without a separate `created_at` index. | — | — |
| 2 | Should `open_db` accept a `&str` key or a `SecretString` type to prevent the key appearing in heap dumps? | — | — |
