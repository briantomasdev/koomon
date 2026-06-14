# Koomon — System Architecture

This document is the authoritative technical specification for Koomon's cross-platform engineering core. It covers the component topology, platform adapters, inter-process communication contracts, and the mobile background lifecycle strategy.

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [High-Level Component Topology](#2-high-level-component-topology)
3. [Technical Component Specifications](#3-technical-component-specifications)
   - 3.1 [Shared Rust Core](#31-shared-rust-core)
   - 3.2 [Desktop Adapter (Tauri Desktop)](#32-desktop-adapter-tauri-desktop)
   - 3.3 [Mobile Adapter (Tauri Mobile)](#33-mobile-adapter-tauri-mobile)
4. [Native Mobile Background Lifecycle](#4-native-mobile-background-lifecycle)
5. [Data Layer](#5-data-layer)
6. [Network Layer](#6-network-layer)
7. [Security Model](#7-security-model)
8. [Dependency Registry](#8-dependency-registry)

---

## 1. Design Philosophy

Koomon's architecture is built around three non-negotiable constraints:

| Constraint | Implication |
|---|---|
| **Local-first** | Every read/write hits a local SQLite database first. The network is a sync mechanism, not a requirement. |
| **Single engineer maintainable** | The UI layer and the sync logic are strictly decoupled. One Rust engineer can maintain core parity across all five platforms. |
| **Privacy by default** | End-to-end encryption is a property of the CRDT message envelope, not a bolt-on feature. The server is a relay, never an authority. |

---

## 2. High-Level Component Topology

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

### Layer Responsibilities

| Layer | Runtime | Responsibility |
|---|---|---|
| React/TypeScript SPA | Host WebView | Render UI state, dispatch user intent |
| Tauri IPC Bridge | Native process boundary | Serialize/deserialize commands and events between WebView and Rust |
| Rust Core | Native binary | CRDT merge, local persistence, transport management |
| Platform Adapters | OS-specific | Lifecycle management, push notification bindings, daemon registration |

---

## 3. Technical Component Specifications

### 3.1 Shared Rust Core

The Rust core is compiled as a single Cargo workspace library (`koomon-core`) that is linked into every platform adapter. It has **zero platform-specific code**.

#### State Synchronization — Y-CRDT

- Crate: [`yrs`](https://crates.io/crates/yrs) (the canonical Rust port of Y.js / Yjs)
- Document model: Each conversation thread is a `yrs::Doc` containing a `Text` or `Array` CRDT type.
- Offline behavior: Local mutations apply immediately to the in-memory CRDT document. The resulting `Update` byte sequence is persisted to SQLite as a pending operation log entry.
- Merge semantics: On reconnect, the client sends all pending `Update` vectors to the server, which fans them out to connected peers. The CRDT guarantees convergence regardless of update arrival order.
- Alternative: [`automerge-rs`](https://crates.io/crates/automerge) is a viable drop-in if richer map/list semantics are required in Phase 2.

#### Local Persistence — SQLite via rusqlite

- Crate: [`rusqlite`](https://crates.io/crates/rusqlite) with the `bundled` feature (statically links SQLite 3.x).
- Schema strategy: One SQLite database file per Koomon workspace, stored in the OS-appropriate application data directory.
- Tables:
  - `messages` — immutable append-only CRDT update log (id, channel_id, author_id, crdt_update BLOB, vector_clock, created_at)
  - `channels` — channel metadata (id, name, workspace_id, type)
  - `peers` — known peer identities and their last-seen vector clocks
  - `pending_sync` — outbound updates not yet acknowledged by the server relay

#### Network Communication — WebTransport / QUIC

- Crate: [`wtransport`](https://crates.io/crates/wtransport)
- Protocol: HTTP/3 over QUIC (UDP). Each client opens a single `Session` to the Koomon relay server, then multiplexes per-channel bidirectional `Stream` objects within that session.
- Benefits over WebSocket: Head-of-line blocking elimination, native stream multiplexing, built-in TLS 1.3, lower latency on unreliable networks (mobile handoff, Wi-Fi ↔ LTE).
- Firewall consideration: QUIC runs on UDP. Self-hosted deployments must open the configured UDP port (default `4433`). The `docker-compose.yml` handles this automatically.

---

### 3.2 Desktop Adapter (Tauri Desktop)

- Framework: [Tauri v2](https://tauri.app/)
- Targets: macOS (arm64 + x86_64), Windows (x86_64), Linux (x86_64, aarch64)
- WebView engine: WKWebView (macOS), WebView2 (Windows), WebKitGTK (Linux)

#### Persistent Background Daemon

On desktop, the Rust core runs as a persistent OS process registered with the host system tray/menubar. This gives Koomon:

- **Always-on WebTransport connection**: The QUIC session remains open even when the UI window is closed.
- **Instant notification delivery**: Incoming CRDT updates trigger a native OS notification without requiring the user to have the app window focused.
- **Low resource footprint**: The Tauri shell compiles to ~5–15 MB on disk. There is no Electron Chromium bundle.

#### IPC Contract

Tauri commands (Rust `#[tauri::command]`) expose the core engine to the TypeScript frontend:

```
invoke("send_message", { channelId, content })  →  ()
invoke("get_channel_history", { channelId })     →  Message[]
invoke("sync_state", {})                          →  SyncStatus
listen("new_message", handler)                   →  UnlistenFn
listen("peer_joined", handler)                   →  UnlistenFn
```

---

### 3.3 Mobile Adapter (Tauri Mobile)

- Framework: [Tauri v2 Mobile](https://v2.tauri.app/start/create-project/#mobile)
- Targets: iOS (arm64), Android (arm64-v8a, armeabi-v7a, x86_64)

#### Platform Plugins

Mobile-specific capabilities are implemented as Tauri plugins backed by native Swift/Kotlin code:

| Plugin | Platform | Responsibility |
|---|---|---|
| `tauri-plugin-apns` | iOS (Swift) | Register device token with APNs, receive silent `content-available` push payloads, wake Rust core |
| `tauri-plugin-fcm` | Android (Kotlin) | Register FCM token, receive high-priority data-only FCM messages, wake Rust core |
| `tauri-plugin-notifications` | Both | Emit local OS notification after CRDT sync completes in background |

#### Foreground vs. Background Modes

| App State | Sync Mechanism | Connection Type |
|---|---|---|
| **Foreground** | Active WebTransport stream | Persistent QUIC session (same as desktop) |
| **Background / Minimized** | Silent push → ephemeral wake | APNs/FCM data payload → transient Rust execution → process suspend |

---

## 4. Native Mobile Background Lifecycle

Mobile operating systems aggressively terminate background processes. Koomon routes around this by converting server events into transient client executions triggered by silent push notifications.

```
[Mobile Client (Background)]  [APNs / FCM]    [Koomon Server]   [Peer Client]
         |                         |                  |                |
         |                         |                  |<--Sends Msg--->|
         |                         |                  |                |
         |                         |<--Trigger Push---|                |
         |                         |  (Data Payload)  |                |
         |<--Deliver Silent Push---|                  |                |
         |                                            |                |
 [Wakes Native Swift/Kotlin Plugin]                  |                |
 [Initializes Rust Core Engine]                      |                |
         |                                            |                |
         |============Connect via WebTransport=======>|                |
         |<===========Sync Outbound/Inbound CRDTs====|                |
         |                                            |                |
 [Commits delta to Local SQLite]                     |                |
 [Emits Local OS Notification]                       |                |
         |                                            |                |
 [Process Suspended by OS]                           |                |
         v                                            v                v
```

### iOS Constraints (APNs)

- Silent push (`content-available: 1`) grants approximately **30 seconds** of background CPU time.
- The Rust core must complete WebTransport connect + CRDT sync + SQLite commit within this window.
- Background App Refresh must be enabled by the user. Koomon's onboarding flow requests this permission explicitly.

### Android Constraints (FCM)

- High-priority FCM data messages wake a `FirebaseMessagingService` regardless of Doze mode.
- The Kotlin plugin spawns a `WorkManager` task with the `EXPEDITED` flag to run the Rust core.
- The execution window is effectively unbounded for expedited work, but Koomon targets < 5 seconds for user-perceived notification latency.

---

## 5. Data Layer

### Local Storage Paths

| Platform | SQLite Path |
|---|---|
| macOS | `~/Library/Application Support/koomon/<workspace-id>.db` |
| Windows | `%APPDATA%\koomon\<workspace-id>.db` |
| Linux | `~/.local/share/koomon/<workspace-id>.db` |
| iOS | App's Documents sandbox |
| Android | App's internal storage |

### Encryption at Rest

- SQLite databases are encrypted using [SQLCipher](https://www.zetetic.net/sqlcipher/) (via the `rusqlite` `sqlcipher` feature).
- The encryption key is derived from the user's device credential store (Keychain on macOS/iOS, DPAPI on Windows, Secret Service on Linux, Android Keystore on Android).

---

## 6. Network Layer

### Server Relay

The Koomon server is a stateless QUIC relay. It:

1. Authenticates peers via JWT issued at workspace join time.
2. Routes CRDT `Update` packets between members of the same channel.
3. Buffers undelivered updates for offline peers (up to a configurable retention window).
4. Does **not** store message plaintext — only encrypted CRDT update blobs.

### Self-Hosted Deployment

```yaml
# docker-compose.yml (simplified)
services:
  koomon-server:
    image: ghcr.io/your-org/koomon-server:latest
    ports:
      - "4433:4433/udp"   # WebTransport / QUIC
      - "8080:8080/tcp"   # HTTP admin API
    volumes:
      - koomon-data:/var/lib/koomon
    environment:
      - KOOMON_JWT_SECRET=${JWT_SECRET}
      - KOOMON_BUFFER_TTL=7d
```

---

## 7. Security Model

| Property | Mechanism |
|---|---|
| Transport security | TLS 1.3 (mandatory, enforced by QUIC) |
| Message confidentiality | End-to-end encrypted CRDT envelopes (NaCl box / X25519 + XSalsa20-Poly1305) |
| Authentication | Ed25519 device identity keys; JWT relay tokens |
| Server trust | Server sees ciphertext only; cannot read message content |
| Local data | SQLCipher AES-256 at rest, key in OS credential store |

---

## 8. Dependency Registry

| Crate / Package | Version Target | Purpose |
|---|---|---|
| `yrs` | ≥ 0.21 | Y-CRDT document engine |
| `rusqlite` | ≥ 0.31 (bundled) | Embedded SQLite storage |
| `wtransport` | ≥ 0.5 | WebTransport / HTTP3 / QUIC client-server |
| `tokio` | ≥ 1.37 | Async runtime |
| `serde` / `serde_json` | ≥ 1.0 | Serialization |
| `tauri` | v2 | Cross-platform app shell + IPC |
| `react` | ≥ 18 | UI framework |
| `typescript` | ≥ 5.4 | Type-safe frontend |
