# Firefly Pay

> Payments that hop across nearby phones until one finds internet.

A Spring Boot backend for **offline UPI-style payments routed through a Bluetooth-style mesh network**. A payment is encrypted on the sender's phone, passed device-to-device through strangers' phones, and uploaded by whichever phone reaches the internet first. The backend decrypts, deduplicates and settles it **exactly once**.

This repo is the server side, plus a software simulator of the mesh, so the whole flow runs on one laptop with no Bluetooth hardware.

## Features

- **End-to-end encryption through untrusted relays**: hybrid RSA-OAEP + AES-256-GCM; intermediaries can't read or modify a packet
- **Exactly-once settlement**: atomic idempotency check on the ciphertext's SHA-256 hash, so a packet delivered by several bridges at once settles only once
- **Tamper and replay protection**: GCM auth tag, signed timestamp (24h window) and per-payment nonce
- **Consistent balances**: transactional debit/credit, optimistic locking (`@Version`), and a unique index on the packet hash as a database-level backstop
- **Interactive dashboard** to inject payments, run gossip rounds and flush bridges

## Quick start

**Requires:** JDK 17+

```bash
git clone https://github.com/srivarshithagoli/firefly-pay.git
cd firefly-pay
./mvnw spring-boot:run        # Windows: mvnw.cmd spring-boot:run
```

Open **http://localhost:8080**. Run the tests with `./mvnw test`.

## Demo flow

1. **Compose** a payment (sender, receiver, amount) and inject it into the mesh
2. **Run gossip rounds**: the packet spreads across virtual phones, TTL drops each hop
3. **Flush bridges**: the phone with internet uploads its packets to the backend
4. Watch **balances** and the **ledger** update

## How it works

```mermaid
flowchart LR
    A[Sender phone offline] -->|encrypt and gossip| B[Stranger phones]
    B --> C[Bridge phone with 4G]
    C -->|HTTPS POST| D[Ingest endpoint]
    D --> E[Hash ciphertext SHA-256]
    E --> F[Claim hash atomically]
    F --> G[Decrypt RSA-OAEP and AES-GCM]
    G --> H[Freshness check 24h]
    H --> I[Transactional settlement]
```

## Key design decisions

**Untrusted relays.** The sender generates a fresh AES-256 key per packet, encrypts the payload with AES-GCM, and wraps the AES key with the server's RSA public key. GCM is *authenticated* encryption, so flipping any bit makes decryption fail instead of producing garbage.

**Duplicate storm.** If three bridges upload the same packet at the same instant, naive processing debits the sender three times. The server instead calls `putIfAbsent(SHA-256(ciphertext))` first; it's atomic, so exactly one thread wins and the rest are dropped as `DUPLICATE_DROPPED` before any decryption happens.
- *Why hash the ciphertext?* `packetId` can be rewritten by a relay, and the plaintext would need an expensive RSA decrypt first. Identical deliveries have byte-identical ciphertext.

**Replay attacks.** The signed timestamp and a UUID nonce live *inside* the encrypted payload, so an attacker can't alter them. Two legitimate payments of the same amount get different nonces and both settle; a replayed packet is byte-identical and is caught by the idempotency check.

## API

| Method | Path | Description |
|---|---|---|
| POST | `/api/bridge/ingest` | Bridge uploads a packet (the production endpoint) |
| POST | `/api/demo/send` | Simulate a sender phone: encrypt and inject a packet |
| POST | `/api/mesh/gossip` | Run one gossip round |
| POST | `/api/mesh/flush` | Bridges with internet upload their packets |
| POST | `/api/mesh/reset` | Clear mesh and idempotency cache |
| GET | `/api/accounts` | Balances |
| GET | `/api/transactions` | Last 20 settled transactions |
| GET | `/api/mesh/state` | State of every virtual device |
| GET | `/api/server-key` | Server RSA public key |

`/api/bridge/ingest` response:

```json
{ "outcome": "SETTLED", "packetHash": "a3f8c9...", "reason": null, "transactionId": 42 }
```

`outcome` is one of `SETTLED`, `DUPLICATE_DROPPED`, `INVALID`.

## Tests

- `encryptDecryptRoundTrip`: hybrid encryption is symmetric
- `tamperedCiphertextIsRejected`: a flipped byte returns `INVALID` and nothing settles
- `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`: 3 concurrent threads, 1 packet; asserts 1 `SETTLED`, 2 `DUPLICATE_DROPPED`, and the sender debited exactly once

## Limitations and production notes

**Inherent to the concept** (no internet anywhere in the chain):
- The receiver can't verify the sender has funds, so an offline payment is an IOU until settlement. Real offline UPI (UPI Lite) uses a pre-funded, hardware-backed wallet for this reason.
- A malicious sender can double-spend offline; whichever packet settles first wins and the other is `REJECTED`.
- Real BLE between strangers' phones is hard (Android background limits, iOS restrictions). The mesh here is simulated.

**What would change for production:**

| Demo | Production |
|---|---|
| H2 in-memory DB | PostgreSQL |
| `ConcurrentHashMap` idempotency | Redis `SET NX EX` |
| RSA keypair generated at startup | Private key in an HSM / KMS |
| Simulated mesh | Real BLE / Wi-Fi Direct |
| No auth on ingest endpoint | Mutual TLS or signed bridge certificates |
| No rate limiting | Per-bridge and per-sender limits |

## Tech stack

Java 17, Spring Boot 3.3, Spring Data JPA / Hibernate, H2, JUnit, Maven

## Project structure

```
src/main/java/com/demo/upimesh/
├── model/        JPA entities + wire formats (Account, Transaction, MeshPacket)
├── crypto/       RSA keypair + hybrid encryption (HybridCryptoService)
├── service/      Ingestion pipeline, idempotency, settlement, mesh simulator
├── controller/   REST endpoints + dashboard
└── config/
src/test/         Concurrency, tamper and crypto tests
```

