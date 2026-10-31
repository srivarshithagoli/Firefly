# 09 — Security, Reliability, and Threat Analysis

This document investigates validation, authentication, authorization, packet integrity, hashing, replay protection, idempotency, concurrency, error handling, and sensitive-data handling, clearly distinguishing what is implemented versus what is absent.

---

## 1. Security Measures Implemented

- **Packet Integrity & Confidentiality**:
  - Encrypted end-to-end using Hybrid RSA-OAEP + AES-256-GCM (`HybridCryptoService.java`).
  - **Authenticated Encryption (GCM)**: If an intermediary alters even a single bit of the ciphertext, decryption fails and throws an exception (`tamperedCiphertextIsRejected` test).
- **Replay Protection**:
  - `PaymentInstruction` includes a `signedAt` timestamp and a unique `nonce`.
  - `BridgeIngestionService` rejects packets older than 86,400 seconds (`upi.mesh.packet-max-age-seconds`).
- **Idempotency & Duplicate Prevention**:
  - `IdempotencyService` uses `ConcurrentHashMap.putIfAbsent()` on the SHA-256 hash of the ciphertext.
  - Database-level unique index on `transactions.packetHash` acts as a defense-in-depth fallback.
- **Concurrency & Loss Prevention**:
  - `@Transactional` ensures ACID guarantees on settlements.
  - `@Version` on `Account` provides optimistic locking to prevent race conditions.
- **Input Validation**:
  - `@NotBlank`, `@Min` annotations on `MeshPacket.java`.

---

## 2. What is Insufficient or Simulated

- **Key Management**:
  - The RSA private key is generated fresh in memory on every startup (`ServerKeyHolder.java`). In production, this must reside in a Hardware Security Module (HSM) or KMS (AWS KMS, Vault).
- **Bridge Authentication**:
  - The endpoint `/api/bridge/ingest` reads `X-Bridge-Node-Id` from HTTP headers without cryptographic verification or Mutual TLS (mTLS). Any client can spoof a bridge node ID.
- **Account & PIN Validation**:
  - Accounts are stored locally in H2 with a hardcoded PIN hash. Real UPI transactions require communicating with a banking core and verifying PINs against bank records.

---

## 3. What is Absent

- **Rate Limiting & DoS Protection**: There is no rate limiter on `/api/bridge/ingest`, leaving the server vulnerable to packet flood attacks.
- **Auditing & SIEM Logging**: Logs go to the standard console without structured SIEM integration or intrusion detection alerts.
- **Offline Double-Spend Prevention**: Because transactions are deferred until a bridge reaches internet, a sender could theoretically spend the same balance across multiple isolated mesh networks before the first packet settles (inherent limitation of deferred mesh settlement).
