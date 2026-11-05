# UPI Offline Mesh — Deferred Settlement over a Device-to-Device Mesh

A Spring Boot backend that demonstrates **offline payments carried through a Bluetooth-style mesh network and settled later, exactly once**.

> You are in a basement with no connectivity. You pay a friend ₹500. Your phone encrypts the payment and hands it to nearby phones. The packet hops device to device until *some* phone walks outside, gets 4G, and uploads it. The backend decrypts, de-duplicates and settles it.

This repository is the **server side** of that system plus a **software simulator of the mesh**, so the entire flow runs on one laptop with no Bluetooth hardware.

> **Honest naming:** this is *mesh-routed deferred settlement*, not real-time offline UPI. The receiver cannot know the money exists until the packet reaches the server. See [Limitations](#limitations-of-the-concept) and [improvement.md](improvement.md).

---

## Contents

1. [What it demonstrates](#what-it-demonstrates)
2. [Tech stack](#tech-stack)
3. [Quick start](#quick-start)
4. [Using the dashboard](#using-the-dashboard)
5. [Architecture](#architecture)
6. [The ingestion pipeline](#the-ingestion-pipeline)
7. [Packet and wire formats](#packet-and-wire-formats)
8. [Three hard problems and how they are handled](#three-hard-problems-and-how-they-are-handled)
9. [Project structure](#project-structure)
10. [Data model](#data-model)
11. [API reference](#api-reference)
12. [Configuration](#configuration)
13. [Tests](#tests)
14. [Demo vs. production](#demo-vs-production)
15. [Known issues](#known-issues)
16. [Limitations of the concept](#limitations-of-the-concept)
17. [Troubleshooting](#troubleshooting)

---

## What it demonstrates

| # | Claim | Mechanism |
|---|-------|-----------|
| 1 | Untrusted relays cannot read or alter a payment | Hybrid RSA-OAEP + AES-256-GCM encryption |
| 2 | The same payment delivered by several bridges at once settles **once** | Atomic claim on a hash of the ciphertext (`putIfAbsent`) |
| 3 | Tampered, junk or stale packets are rejected before touching the ledger | GCM auth tag, freshness window, DB transaction |

Claims 1 and 3 are solid. Claim 2 holds for byte-identical deliveries but has a known bypass (see [Known issues](#known-issues)).

---

## Tech stack

- **Java 25 (LTS)**
- **Spring Boot 4.1.2** (Spring Framework 7.0, Tomcat 11): Web MVC, Data JPA (Hibernate ORM 7.4), Validation, Thymeleaf. All versions are managed by the Spring Boot BOM.
- **H2 2.4** in-memory database (zero setup), with the Boot 4 `spring-boot-h2console` module for the web console
- **JCA/JCE** (`javax.crypto`) for RSA-OAEP and AES-GCM, no third-party crypto
- **JUnit Jupiter 6** (the version Boot 4.1 manages) + Mockito 5 + `@SpringBootTest`
- Vanilla JS dashboard (no build step)
- **Maven Wrapper**: no Maven install needed

---

## Quick start

### Prerequisites
- **JDK 25** on `PATH` or `JAVA_HOME` (check with `java -version`). Spring Boot 4.1 and the build target Java 25; JDK 17 and 21 will **not** compile it.
- Nothing else: no database, no Redis, no Maven install (the wrapper downloads Maven 3.9.9)

### Run

Windows (PowerShell):
```powershell
cd UPI_Without_Internet
.\mvnw.cmd spring-boot:run
```

macOS / Linux:
```bash
./mvnw spring-boot:run
```

The first run downloads Maven (~10 MB) and dependencies (~80 MB); later starts take a few seconds. When you see `Started UpiMeshApplication`, open:

- **Dashboard:** http://localhost:8080
- **H2 console:** http://localhost:8080/h2-console (JDBC URL `jdbc:h2:mem:upimesh`, user `sa`, empty password)

Stop with `Ctrl+C`.

### Build a jar
```powershell
.\mvnw.cmd clean package
java -jar target\upi-offline-mesh-0.0.1-SNAPSHOT.jar
```

### Test
```powershell
.\mvnw.cmd test
```

---

## Using the dashboard

Seeded accounts: `alice@demo` ₹5000, `bob@demo` ₹1000, `carol@demo` ₹2500, `dave@demo` ₹500.

Seeded mesh: `phone-alice`, `phone-stranger1..3` (offline) and `phone-bridge` (**has internet**).

### Happy path
1. **Compose and inject.** Pick sender, receiver, amount, PIN, then *Inject into Mesh*. The server plays the sender's phone: it builds a `PaymentInstruction`, encrypts it with its own public key and hands the packet to `phone-alice`.
2. **Run gossip** once or twice. Every device copies every packet it holds to every other device, TTL minus 1 per hop.
3. **Bridges upload.** `phone-bridge` POSTs everything it holds to the ingest pipeline. Balances and the ledger update.

### Idempotency, three ways
- **Re-flush:** click *Bridges Upload* a second time. The same packets are re-sent and come back `DUPLICATE_DROPPED`; balances do not move again.
- **Concurrent:** `.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` fires 3 threads at one packet; exactly one settles.
- **Multi-bridge:** to see parallel duplicates through the UI, mark more devices as `hasInternet=true` in `MeshSimulatorService.seedDefaultDevices()`.

### Reset
*Reset* clears the mesh and the idempotency cache. **It does not restore balances or the ledger**; restart the app for a clean state.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                       SENDER PHONE (offline)                         │
│  PaymentInstruction {sender, receiver, amount, pinHash, nonce, time} │
│        │  encrypt with server RSA public key (hybrid)                │
│        ▼                                                             │
│  MeshPacket {packetId, ttl, createdAt, ciphertext}                   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ BLE gossip (simulated)
                               ▼
     ┌──────────┐  hop  ┌──────────┐  hop  ┌──────────┐
     │stranger 1│ ────▶ │stranger 2│ ────▶ │  bridge  │ ◀─ walks outside, gets 4G
     └──────────┘       └──────────┘       └────┬─────┘
                                                │ HTTPS POST /api/bridge/ingest
                                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      SPRING BOOT BACKEND                             │
│                                                                      │
│  [1] SHA-256(ciphertext string)  ───────────► packetHash             │
│  [2] IdempotencyService.claim()  ── atomic putIfAbsent ─► DUPLICATE  │
│  [3] HybridCryptoService.decrypt ── RSA-OAEP + AES-GCM ─► INVALID    │
│  [4] freshness: -5 min ≤ age ≤ 24 h ────────────────────► INVALID    │
│  [5] SettlementService.settle() ── @Transactional ─► SETTLED/REJECTED│
│         debit + credit + ledger row, @Version optimistic lock        │
└──────────────────────────────────────────────────────────────────────┘
```

Layers: `controller` (HTTP) → `service` (pipeline, simulator, settlement) → `crypto` and `model` (JPA).

---

## The ingestion pipeline

`BridgeIngestionService.ingest(packet, bridgeNodeId, hopCount)` returns one of:

| Outcome | Meaning |
|---------|---------|
| `SETTLED` | Debit and credit committed, ledger row written (`transactionId` set) |
| `DUPLICATE_DROPPED` | Hash already claimed; nothing done |
| `INVALID` | `decryption_failed`, `stale_packet`, `future_dated`, or `internal_error: …` |

`SettlementService.settle` can also record a ledger row with status `REJECTED` (insufficient balance). The API still reports that as outcome `SETTLED` with a `transactionId`; check the ledger row's `status`. (See [improvement.md](improvement.md).)

The endpoint always answers HTTP 200; the outcome is in the body.

---

## Packet and wire formats

### Outer `MeshPacket` (visible to every relay)
```json
{ "packetId": "uuid", "ttl": 5, "createdAt": 1730000000000, "ciphertext": "<base64>" }
```
`packetId`, `ttl` and `createdAt` are **not authenticated**; relays could rewrite them. The server never trusts them for money decisions.

### Ciphertext layout (before base64)
```
[ 256 B  RSA-OAEP(SHA-256, MGF1) encrypted AES-256 key ]
[  12 B  GCM IV                                        ]
[  N + 16 B  AES-256-GCM ciphertext || 128-bit tag     ]
```
A fresh AES key and IV are generated per packet. Any bit flip causes GCM tag verification to fail, so decryption throws.

### Inner `PaymentInstruction` (only the server can read)
`senderVpa`, `receiverVpa`, `amount`, `pinHash` (SHA-256 of the PIN), `nonce` (UUID), `signedAt` (epoch ms).

---

## Three hard problems and how they are handled

### 1. Untrusted intermediaries
Relays are strangers' phones. Hybrid encryption means they only ever see opaque bytes, and GCM authentication means they cannot modify them undetected. RSA alone is too small for the payload, so RSA wraps a one-time AES key (the same pattern as TLS and PGP).

### 2. The duplicate storm
Three bridges hold the same packet and upload within milliseconds. The first step is to hash the ciphertext and *claim* the hash:

```java
Instant prev = seen.putIfAbsent(packetHash, now);   // atomic
return prev == null;                                // true = first, false = duplicate
```

`ConcurrentHashMap.putIfAbsent` is atomic, so exactly one caller wins. Only the winner decrypts and settles. In production this becomes Redis `SET key NX EX`. A unique index on `transactions.packet_hash` is a second line of defence.

Why hash the ciphertext, not `packetId`? `packetId` is rewritable by relays. Why not the plaintext? Because that needs an expensive RSA decrypt first.

### 3. Replay
`signedAt` is inside the encrypted payload, so the server rejects anything older than 24 h (and anything more than 5 min in the future). Replaying a byte-identical packet is caught by the idempotency claim.

> The `nonce` is carried in the payload but **is not currently checked or stored**. Uniqueness today comes only from the ciphertext hash. See [improvement.md](improvement.md) #1 to #3.

---

## Project structure

```
UPI_Without_Internet/
├── pom.xml                         Spring Boot 4.1.2, Java 25
├── mvnw, mvnw.cmd, .mvn/           Maven wrapper
├── README.md, improvement.md
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties
    │   │   └── templates/dashboard.html          Single-page demo UI
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java
    │       ├── config/AppConfig.java             @EnableScheduling
    │       ├── controller/
    │       │   ├── ApiController.java            All REST endpoints
    │       │   └── DashboardController.java      Serves "/"
    │       ├── crypto/
    │       │   ├── ServerKeyHolder.java          RSA-2048 keypair at startup (in memory)
    │       │   └── HybridCryptoService.java      encrypt / decrypt / hashCiphertext
    │       ├── model/
    │       │   ├── Account.java                  vpa, holderName, balance, @Version
    │       │   ├── Transaction.java              ledger; unique packetHash
    │       │   ├── MeshPacket.java               wire format
    │       │   ├── PaymentInstruction.java       decrypted payload
    │       │   └── *Repository.java              Spring Data JPA
    │       └── service/
    │           ├── BridgeIngestionService.java   The pipeline
    │           ├── IdempotencyService.java       ConcurrentHashMap claim + TTL eviction
    │           ├── SettlementService.java        @Transactional debit/credit/ledger
    │           ├── DemoService.java              Seeds accounts; plays the sender phone
    │           ├── MeshSimulatorService.java     Gossip across VirtualDevices
    │           └── VirtualDevice.java            One simulated phone
    └── test/java/com/demo/upimesh/
        └── IdempotencyConcurrencyTest.java       3 tests
```

---

## Data model

**accounts**: `vpa` (PK), `holder_name`, `balance` `DECIMAL(19,2)`, `version` (optimistic lock).

**transactions**: `id`, `packet_hash` (unique, 64 hex), `sender_vpa`, `receiver_vpa`, `amount`, `signed_at`, `settled_at`, `bridge_node_id`, `hop_count`, `status` (`SETTLED` | `REJECTED`).

Schema is auto-created (`ddl-auto=create-drop`); all data is lost on shutdown.

---

## API reference

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/` | Dashboard |
| GET | `/api/server-key` | Server RSA public key (base64 X.509) |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Latest 20 ledger rows |
| GET | `/api/mesh/state` | Every virtual device and the packets it holds |
| POST | `/api/demo/send` | Simulate a sender: build + encrypt + inject a packet |
| POST | `/api/mesh/gossip` | One gossip round |
| POST | `/api/mesh/flush` | All internet-enabled devices upload in parallel |
| POST | `/api/mesh/reset` | Clear devices and idempotency cache |
| POST | `/api/bridge/ingest` | **The production-shaped endpoint** |
| GET | `/h2-console` | DB browser |

### `POST /api/demo/send`
```json
{ "senderVpa": "alice@demo", "receiverVpa": "bob@demo", "amount": 500, "pin": "1234", "ttl": 5, "startDevice": "phone-alice" }
```

### `POST /api/bridge/ingest`
```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{ "packetId": "550e8400-...", "ttl": 2, "createdAt": 1730000000000, "ciphertext": "<base64>" }
```
Response:
```json
{ "outcome": "SETTLED", "packetHash": "a3f8c9…", "reason": null, "transactionId": 42 }
```

Try it with curl:
```bash
curl -s localhost:8080/api/accounts
curl -s -X POST localhost:8080/api/demo/send -H 'Content-Type: application/json' \
  -d '{"senderVpa":"alice@demo","receiverVpa":"bob@demo","amount":100,"pin":"1234"}'
curl -s -X POST localhost:8080/api/mesh/gossip
curl -s -X POST localhost:8080/api/mesh/flush
```

---

## Configuration

`src/main/resources/application.properties`

| Property | Default | Meaning |
|----------|---------|---------|
| `server.port` | `8080` | HTTP port |
| `spring.datasource.url` | `jdbc:h2:mem:upimesh` | In-memory DB |
| `spring.jpa.hibernate.ddl-auto` | `create-drop` | Recreate schema each run |
| `spring.h2.console.enabled` | `true` | **Disable outside demos** |
| `upi.mesh.idempotency-ttl-seconds` | `86400` | How long a packet hash is remembered |
| `upi.mesh.packet-max-age-seconds` | `86400` | Freshness window |

Keep the idempotency TTL **≥** the freshness window; otherwise an expired hash lets a still-fresh packet through again.

### Migration notes (Spring Boot 3.3.5 / Java 17 → 4.1.2 / Java 25)

- `spring-boot-starter-web` → `spring-boot-starter-webmvc` (Boot 4 modular starters).
- The H2 web console now needs the `spring-boot-h2console` dependency (added, runtime scope).
- Removed `spring.jpa.properties.hibernate.dialect`: Hibernate 7 picks the H2 dialect itself and warned that the setting is redundant.
- Spring MVC now serialises with **Jackson 3** (`tools.jackson`); REST payloads are unchanged. `HybridCryptoService` still uses Jackson 2 (`com.fasterxml`) through the explicit `jackson-databind` dependency for the packet payload. Moving it to Jackson 3 is a follow-up, not required yet.
- Application code, entities, endpoints and tests needed **no changes**.

---

## Tests

`IdempotencyConcurrencyTest` (Spring Boot context, H2):

| Test | Asserts |
|------|---------|
| `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` | 3 threads, 1 packet → 1 `SETTLED`, 2 `DUPLICATE_DROPPED`, balances move once |
| `tamperedCiphertextIsRejected` | One flipped character → `INVALID`, no crash |
| `encryptDecryptRoundTrip` | Encrypt → decrypt preserves fields |

Not covered yet: insufficient funds, unknown VPA, stale / future packets, base64-variant duplicates, controller layer, restart behaviour. See [improvement.md](improvement.md).

---

## Demo vs. production

| Demo | Production |
|------|------------|
| H2 in memory | PostgreSQL with migrations (Flyway) |
| `ConcurrentHashMap` idempotency | Redis `SET NX EX`, or DB unique constraint as the source of truth |
| RSA key regenerated at every start | KMS/HSM-held key, key IDs, rotation |
| Server builds packets (`DemoService`) | Same logic in an Android/iOS client |
| Simulated mesh | Real BLE GATT / Wi-Fi Direct |
| Server owns the ledger | Integration with a bank core / NPCI |
| Open `/api/bridge/ingest` | mTLS or signed bridge certificates |
| Unverified PIN | Bank-side PIN verification / device-bound signature |
| H2 console exposed, no rate limits | Disabled; per-node and per-sender limits |

---

## Known issues

These are real defects found by reading the code, not theoretical ones. Full analysis, impact and fixes are in **[improvement.md](improvement.md)**. Summary:

1. **No sender authentication.** The server's public key is public, so anyone can forge a valid packet "from" any VPA. The PIN hash is ignored.
2. **Idempotency bypass.** The hash covers the base64 *string*; a relay can change the last character (`QQ==` → `QR==`, same bytes) and get a second settlement.
3. **The nonce is never enforced.**
4. **Claim-before-settle loses payments.** If settlement throws after the claim, the hash stays claimed and the packet can never be retried.
5. **Idempotency is in-memory** and the **RSA key regenerates on restart**: in-flight packets become undecryptable and dedupe state vanishes.
6. **`REJECTED` payments are reported as `SETTLED`** by the API.
7. **No auth on any endpoint**, including `/api/mesh/reset`, `/api/demo/send` and the H2 console.

---

## Limitations of the concept

These come from "no internet anywhere in the chain", not from implementation bugs:

1. **No proof of funds offline.** The receiver holds an IOU. If the sender's balance is short at settlement, the payment is `REJECTED` and the receiver has no recourse. Real offline UPI (UPI Lite) uses a pre-funded, hardware-backed wallet for exactly this reason.
2. **Offline double-spend.** A sender can issue the same funds to two receivers; the first packet to reach the server wins.
3. **BLE is hard in practice.** Android throttles background BLE; iOS restricts peripheral mode. This demo simulates the mesh to avoid the problem.
4. **Metadata and liability.** Strangers carry your encrypted packets; their existence leaks metadata and raises regulatory questions.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `java: command not found` | Install a JDK: `winget install EclipseAdoptium.Temurin.25.JDK` |
| `invalid target release: 25` / `release version 25 not supported` | Your JDK is older than 25. Point `JAVA_HOME` at JDK 25 |
| Port 8080 in use | Change `server.port` |
| `mvnw.cmd` not recognised in PowerShell | Use `.\mvnw.cmd` |
| First run is slow | Maven and dependencies are downloading |
| Balances look wrong after testing | *Reset* does not restore balances; restart the app |
| Concurrency test flakes | Re-run; if it fails consistently, open an issue with the output |

---

## License

Demo / educational code. No license specified.
