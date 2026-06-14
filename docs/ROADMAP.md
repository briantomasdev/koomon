# Koomon — Development Roadmap

This document defines the four-phase MVP execution plan from initial Rust workspace initialization to public open-source launch. Each phase has concrete deliverables, acceptance criteria, and technical milestones.

---

## Table of Contents

1. [Overview Timeline](#1-overview-timeline)
2. [Phase 1 — Local-First Core (Weeks 1–6)](#2-phase-1--local-first-core-weeks-16)
3. [Phase 2 — Network Transport (Weeks 7–12)](#3-phase-2--network-transport-weeks-712)
4. [Phase 3 — Native Mobile Integration (Weeks 13–20)](#4-phase-3--native-mobile-integration-weeks-1320)
5. [Phase 4 — Open-Source Launch (Weeks 21–24)](#5-phase-4--open-source-launch-weeks-2124)
6. [Post-Launch Backlog](#6-post-launch-backlog)

---

## 1. Overview Timeline

```
Week:  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23   24
       ├────────────────────────┤
       │  Phase 1: Local Core   │
                                 ├────────────────────────┤
                                 │  Phase 2: Transport     │
                                                           ├────────────────────────────────┤
                                                           │  Phase 3: Mobile Integration   │
                                                                                             ├────────────────┤
                                                                                             │  Phase 4: Launch│
```

| Phase | Focus | Weeks | Exit Criteria |
|---|---|---|---|
| 1 | Local-First Core | 1–6 | CRDT convergence tests pass on concurrent offline edits |
| 2 | Network Transport | 7–12 | Real-time text sync across 3 local nodes via Tauri desktop |
| 3 | Native Mobile | 13–20 | Background push wakes app and syncs message on locked screen |
| 4 | OSS Launch | 21–24 | Public GitHub repo with passing CI, Docker deploy, and 100+ GitHub stars |

---

## 2. Phase 1 — Local-First Core (Weeks 1–6)

**Objective:** Establish the Rust workspace foundation. Prove that the Y-CRDT engine and SQLite storage layer correctly handle concurrent offline edits and converge to identical states.

### Milestones

| # | Milestone | Target Week |
|---|---|---|
| 1.1 | Initialize Cargo workspace with `koomon-core` library crate | Week 1 |
| 1.2 | Integrate `yrs` (Y-CRDT) — implement `Doc`, `Text`, and `Update` primitives | Week 2 |
| 1.3 | Integrate `rusqlite` (bundled) — implement schema migrations | Week 3 |
| 1.4 | Implement `MessageStore` — append CRDT update blobs, read history | Week 4 |
| 1.5 | Implement `PendingSync` queue — track unacknowledged outbound updates | Week 5 |
| 1.6 | Write automated convergence tests — simulate concurrent offline writes | Week 6 |

### Cargo Workspace Structure

```
koomon/
├── Cargo.toml                  # workspace manifest
├── crates/
│   ├── koomon-core/            # platform-agnostic Rust library
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── crdt/           # Y-CRDT document management
│   │   │   ├── storage/        # rusqlite persistence layer
│   │   │   └── sync/           # pending sync queue
│   │   └── Cargo.toml
│   └── koomon-server/          # relay server binary (stubbed in Phase 1)
│       └── Cargo.toml
```

### Acceptance Criteria

```
GIVEN two clients A and B share a channel document
AND both clients go offline
WHEN A appends "Hello" and B appends "World" concurrently
AND both clients reconnect and exchange CRDT Update vectors
THEN both A and B MUST converge to the identical document state
AND the convergence MUST be deterministic regardless of update delivery order
AND the final state MUST be committed to each client's local SQLite database
```

**Test command:**
```bash
cargo test --package koomon-core -- crdt::convergence
```

### Key Dependencies (Phase 1)

```toml
[dependencies]
yrs = "0.21"
rusqlite = { version = "0.31", features = ["bundled"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

---

## 3. Phase 2 — Network Transport (Weeks 7–12)

**Objective:** Bring the core online. Implement the WebTransport relay server and connect a functional Tauri desktop application to it. Demonstrate real-time text synchronization across multiple local nodes.

### Milestones

| # | Milestone | Target Week |
|---|---|---|
| 2.1 | Integrate `wtransport` into `koomon-server` — accept client QUIC sessions | Week 7 |
| 2.2 | Implement channel-based session routing on the server | Week 8 |
| 2.3 | Implement WebTransport client in `koomon-core` — connect, send, receive | Week 9 |
| 2.4 | Wire `PendingSync` queue to transport layer — auto-flush on reconnect | Week 10 |
| 2.5 | Initialize Tauri desktop app (`koomon-desktop`) — scaffold IPC commands | Week 11 |
| 2.6 | End-to-end integration test: 3 local nodes sync a text document in real time | Week 12 |

### Cargo Workspace Addition (Phase 2)

```
koomon/
├── crates/
│   ├── koomon-core/            # (Phase 1, extended with transport client)
│   ├── koomon-server/          # (Phase 1 stub → full WebTransport relay)
│   └── koomon-desktop/         # NEW: Tauri v2 desktop application
│       ├── src-tauri/
│       │   ├── src/
│       │   │   ├── main.rs
│       │   │   └── commands.rs  # Tauri IPC bridge
│       │   └── Cargo.toml
│       └── src/                 # React/TypeScript SPA
│           ├── App.tsx
│           └── main.tsx
```

### Acceptance Criteria

```
GIVEN a koomon-server instance running on localhost:4433
AND three Tauri desktop clients A, B, C connected to the same channel
WHEN client A types a message
THEN clients B and C MUST receive the CRDT update within 200ms on a LAN
AND all three clients MUST show identical channel state
AND disconnecting client B, sending a message from A, then reconnecting B
MUST result in B receiving the missed update on reconnect
```

**Integration test command:**
```bash
cargo test --package koomon-core -- transport::integration
```

**Run server locally:**
```bash
cargo run --package koomon-server -- --port 4433
```

**Run desktop app:**
```bash
cargo tauri dev --config crates/koomon-desktop/src-tauri/tauri.conf.json
```

### Key Dependencies (Phase 2 additions)

```toml
[dependencies]
wtransport = "0.5"
tauri = { version = "2", features = [] }
tauri-build = { version = "2", features = [] }
```

---

## 4. Phase 3 — Native Mobile Integration (Weeks 13–20)

**Objective:** Extend the Tauri application to iOS and Android. Implement platform-specific Swift and Kotlin plugins to wire APNs and FCM silent push notifications to the Rust core, enabling background sync on locked mobile devices.

### Milestones

| # | Milestone | Target Week |
|---|---|---|
| 3.1 | Initialize Tauri Mobile — configure iOS and Android targets | Week 13 |
| 3.2 | Implement `tauri-plugin-apns` (Swift) — APNs device registration, token exchange | Week 14 |
| 3.3 | Implement `tauri-plugin-fcm` (Kotlin) — FCM registration, high-priority data message handler | Week 15 |
| 3.4 | Implement server-side push trigger — emit APNs/FCM payload on inbound CRDT update | Week 16 |
| 3.5 | Implement background Rust core wake on iOS — complete sync within 30s window | Week 17–18 |
| 3.6 | Implement background Rust core wake on Android — `WorkManager` expedited task | Week 18–19 |
| 3.7 | Implement local OS notification after background sync completes | Week 19 |
| 3.8 | End-to-end mobile background test: send message → locked phone shows notification | Week 20 |

### Workspace Addition (Phase 3)

```
koomon/
├── crates/
│   └── koomon-mobile/           # NEW: Tauri v2 mobile application
│       ├── src-tauri/
│       │   ├── src/main.rs
│       │   └── gen/
│       │       ├── apple/        # Xcode project (iOS)
│       │       └── android/      # Gradle project (Android)
├── plugins/
│   ├── tauri-plugin-apns/       # NEW: Swift plugin
│   │   ├── swift/KoomomAPNS.swift
│   │   └── src/lib.rs           # Rust plugin bridge
│   └── tauri-plugin-fcm/        # NEW: Kotlin plugin
│       ├── android/KoomomFCM.kt
│       └── src/lib.rs           # Rust plugin bridge
```

### iOS Background Execution Flow (Swift)

```swift
// KoomomAPNS.swift (simplified)
func application(_ application: UIApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                 fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    // 1. Decode FCM/APNs data payload
    guard let channelId = userInfo["channel_id"] as? String else {
        completionHandler(.noData); return
    }
    // 2. Wake Rust core via FFI
    koomon_core_sync(channelId) { result in
        // 3. Emit local notification
        scheduleLocalNotification(from: result)
        // 4. Signal OS completion within 30s budget
        completionHandler(.newData)
    }
}
```

### Android Background Execution Flow (Kotlin)

```kotlin
// KoomomFCM.kt (simplified)
class KoomomFirebaseService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        if (message.data["type"] == "crdt_update") {
            val workRequest = OneTimeWorkRequestBuilder<KoomomSyncWorker>()
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .setInputData(workDataOf("channel_id" to message.data["channel_id"]))
                .build()
            WorkManager.getInstance(applicationContext).enqueue(workRequest)
        }
    }
}
```

### Acceptance Criteria

```
GIVEN a mobile device (iOS or Android) with Koomon installed
AND the device screen is locked (app in background / suspended)
WHEN a peer sends a message to a shared channel
THEN within 15 seconds the mobile device MUST display a local OS notification
AND when the user unlocks the device and opens Koomon
THEN the full message content MUST be present in local SQLite
AND no message data MUST have been processed by any cloud service
```

### Required Certificates / Credentials (Development)

| Platform | Requirement |
|---|---|
| iOS | Apple Developer account; APNs auth key (`.p8`); provisioning profile with Push Notifications entitlement |
| Android | Firebase project; `google-services.json` in Android module; FCM server key in environment |

---

## 5. Phase 4 — Open-Source Launch (Weeks 21–24)

**Objective:** Harden the repository for public consumption. Write complete user and operator documentation. Publish a production-ready `docker-compose.yml`. Launch publicly and seed the developer community that will feed the Managed Cloud conversion funnel.

### Milestones

| # | Milestone | Target Week |
|---|---|---|
| 4.1 | Write `CONTRIBUTING.md` — branch strategy, PR template, DCO sign-off | Week 21 |
| 4.2 | Write `docs/SELF_HOSTING.md` — full Docker deployment guide with environment variables | Week 21 |
| 4.3 | Write `docs/SECURITY.md` — responsible disclosure policy and security model | Week 21 |
| 4.4 | Produce `docker-compose.yml` for single-command Community Core server deployment | Week 22 |
| 4.5 | Set up GitHub Actions CI — `cargo test`, `cargo clippy`, Tauri desktop build matrix | Week 22 |
| 4.6 | Tag `v0.1.0-alpha` release with pre-built binaries for macOS, Windows, Linux | Week 23 |
| 4.7 | Launch: GitHub public, Hacker News "Show HN", Reddit r/selfhosted, r/rust | Week 24 |
| 4.8 | Open Managed Cloud waitlist (landing page + email capture) | Week 24 |

### `docker-compose.yml` Target (Phase 4)

```yaml
version: "3.9"

services:
  koomon-server:
    image: ghcr.io/your-org/koomon-server:0.1.0
    restart: unless-stopped
    ports:
      - "4433:4433/udp"    # WebTransport / QUIC
      - "8080:8080/tcp"    # Admin HTTP API
    volumes:
      - koomon-data:/var/lib/koomon
    environment:
      KOOMON_JWT_SECRET: "${JWT_SECRET}"
      KOOMON_DOMAIN: "${DOMAIN}"
      KOOMON_BUFFER_TTL: "7d"
      KOOMON_MAX_WORKSPACES: "10"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  koomon-data:
```

### GitHub Actions CI Matrix (Phase 4)

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    
steps:
  - uses: actions/checkout@v4
  - uses: dtolnay/rust-toolchain@stable
  - run: cargo test --workspace
  - run: cargo clippy --workspace -- -D warnings
  - run: cargo build --release --package koomon-server
```

### Launch Checklist

- [ ] `README.md` with architecture diagram, quick-start, and license clarity
- [ ] `docs/ARCHITECTURE.md` — complete technical spec *(this document)*
- [ ] `docs/BUSINESS_MODEL.md` — pricing and MRR model
- [ ] `docs/SELF_HOSTING.md` — operator deployment guide
- [ ] `docs/SECURITY.md` — responsible disclosure and threat model
- [ ] `CONTRIBUTING.md` — contribution guide and DCO
- [ ] `CHANGELOG.md` — v0.1.0-alpha changes
- [ ] `docker-compose.yml` — production-ready
- [ ] GitHub Actions CI passing on all three OS targets
- [ ] Pre-built binaries attached to `v0.1.0-alpha` release
- [ ] Managed Cloud waitlist landing page live

---

## 6. Post-Launch Backlog

These items are explicitly deferred past Week 24 to protect launch focus:

| Item | Rationale for Deferral |
|---|---|
| SwarmState AI on-device vector indexing | Requires Phase 1–3 corpus to index; drives Enterprise pipeline |
| Koomon Managed Cloud billing integration (Stripe) | Needs paying customers to validate pricing first |
| SSO / SCIM provisioning | Enterprise feature; gated on first Enterprise deal |
| End-to-end encryption key exchange UI | Core crypto is in place; UX polish deferred |
| Windows ARM64 build | Low market priority for initial launch |
| Multi-workspace federation | Complex protocol work; V2 roadmap item |
| Voice / video channels | Requires WebRTC layer; separate engineering initiative |
