# 08 — Core Business Logic & Algorithms

This document explains the main algorithms and business rules in execution order, supported by source code references and beginner-friendly examples.

---

## 1. Hybrid Encryption Algorithm (`HybridCryptoService.java`)
**Problem**: RSA encryption can only encrypt small payloads (~245 bytes for 2048-bit keys), but payment instructions are larger JSON strings. Furthermore, we need fast symmetric encryption combined with asymmetric key distribution.

**Algorithm (Execution Order)**:
1. **Generate Session Key**: A fresh AES-256 key is generated (`KeyGenerator.getInstance("AES")`).
2. **Symmetric Encryption**: The payment instruction JSON object is encrypted using AES-256-GCM (`AES/GCM/NoPadding`) with a random 12-byte IV (Initialization Vector). GCM produces both ciphertext and a 16-byte authentication tag.
3. **Asymmetric Encryption**: The AES session key bytes are encrypted using RSA-OAEP (`RSA/ECB/OAEPWithSHA-256AndMGF1Padding`) with the server's public key.
4. **Wire Assembly**: The components are concatenated: `[256 bytes encrypted AES key][12 bytes IV][AES ciphertext + tag]` and encoded to Base64.

**Analogy**: Placing cash inside a steel box (AES encryption), locking the box with a key, and then locking that key inside a secure safe (RSA encryption) that only the bank manager can open.

---

## 2. Gossip Protocol (`MeshSimulatorService.java`)
**Problem**: Devices are offline in a basement and cannot reach the server directly. How do payments propagate?

**Algorithm (Execution Order)**:
1. **Snapshot**: A snapshot of all virtual devices and their held packets is taken at the start of the round to prevent cascading propagation through all devices in a single step.
2. **Broadcast & TTL Check**: For each device and each packet it holds:
   - If TTL (`packet.getTtl()`) is `<= 0`, the packet is not forwarded further.
   - Otherwise, the packet is copied to every neighboring device that does not already hold it.
   - The copied packet's TTL is decremented by 1 (`ttl - 1`).

---

## 3. Idempotency & Deduplication (`IdempotencyService.java` & `BridgeIngestionService.java`)
**Problem**: Multiple bridge nodes holding the same packet walk outside and upload it simultaneously. How do we prevent duplicate settlement?

**Algorithm (Execution Order)**:
1. **Ciphertext Hashing**: Compute SHA-256 hex of the packet's ciphertext (`HybridCryptoService.hashCiphertext()`).
2. **Atomic Claim**: Attempt to insert the hash into a thread-safe map:
   ```java
   Instant prev = seen.putIfAbsent(packetHash, now);
   return prev == null; // true if first claimer, false if duplicate
   ```
   `ConcurrentHashMap.putIfAbsent` is atomic. If 100 threads attempt to claim the exact same hash simultaneously, exactly one returns `null` (success), and all others return the existing timestamp (duplicate).
3. **Short-Circuit**: Duplicate packets return `DUPLICATE_DROPPED` immediately without decrypting or processing database writes.

---

## 4. Settlement & ACID Transactions (`SettlementService.java`)
**Problem**: Ensuring money is safely and correctly transferred from sender to receiver.

**Algorithm (Execution Order)**:
1. **Database Transaction**: Annotated with `@Transactional`.
2. **Account Retrieval**: Find sender and receiver accounts in the database by VPA.
3. **Validation**: Verify that the transfer amount is positive and the sender has sufficient balance (`sender.getBalance().compareTo(amount) >= 0`). If insufficient, record a `REJECTED` transaction.
4. **Balance Adjustment**:
   - `sender.balance = sender.balance - amount`
   - `receiver.balance = receiver.balance + amount`
5. **Ledger Record**: Save an immutable `Transaction` entity with status `SETTLED` and the packet hash.
