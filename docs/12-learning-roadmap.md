# 12 — Beginner Learning Roadmap

This document provides a prioritized learning plan based on the actual codebase, telling you what to study first, which files to examine, and how to verify your understanding.

---

## Step 1: Understand Spring Boot Basics & Models
- **Why it matters**: Every Spring Boot app starts with data models and basic configuration.
- **Files to Study**:
  1. `src/main/java/com/demo/upimesh/model/Account.java`
  2. `src/main/java/com/demo/upimesh/model/Transaction.java`
  3. `src/main/resources/application.properties`
- **Verification Exercise**: Open `Account.java`, understand JPA annotations (`@Entity`, `@Id`, `@Version`), and check how H2 is configured in `application.properties`.

---

## Step 2: Understand Cryptography & Security
- **Why it matters**: The core innovation of this project is secure offline packet relaying through untrusted intermediaries.
- **Files to Study**:
  1. `src/main/java/com/demo/upimesh/crypto/ServerKeyHolder.java`
  2. `src/main/java/com/demo/upimesh/crypto/HybridCryptoService.java`
- **Verification Exercise**: Read how `HybridCryptoService.encrypt()` combines RSA and AES-GCM, and how `decrypt()` verifies the authentication tag.

---

## Step 3: Understand Business Logic & Ingestion Pipeline
- **Why it matters**: This ties cryptography, idempotency, and database transactions together.
- **Files to Study**:
  1. `src/main/java/com/demo/upimesh/service/IdempotencyService.java`
  2. `src/main/java/com/demo/upimesh/service/SettlementService.java`
  3. `src/main/java/com/demo/upimesh/service/BridgeIngestionService.java`
- **Verification Exercise**: Trace `BridgeIngestionService.ingest()` step by step to see how idempotency checking happens before decryption.

---

## Step 4: Understand Testing & Concurrency
- **Why it matters**: Concurrency testing proves that duplicate bridge uploads don't cause double-spending.
- **Files to Study**:
  1. `src/test/java/com/demo/upimesh/IdempotencyConcurrencyTest.java`
- **Verification Exercise**: Run `mvnw.cmd test` and inspect `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` to see how `CountDownLatch` and `ExecutorService` simulate concurrent bridge uploads.
