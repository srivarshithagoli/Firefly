# 10 — Testing and Debugging Guide

This document explains the existing test suite, how to run tests, how to inspect logs, and how to debug common issues.

---

## Running Tests

To run all unit and integration tests using the Maven wrapper:

- **Windows**:
  ```cmd
  mvnw.cmd test
  ```
- **Mac / Linux**:
  ```bash
  ./mvnw test
  ```

---

## Test Suite Overview (`IdempotencyConcurrencyTest.java`)

The repository includes a Spring Boot test class (`IdempotencyConcurrencyTest.java`) covering critical security and concurrency behaviors:

1. **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`**:
   - The headline test. Fires 3 concurrent threads delivering the same packet simultaneously using an `ExecutorService` and `CountDownLatch`.
   - Asserts that exactly 1 thread succeeds (`SETTLED`) and 2 are dropped (`DUPLICATE_DROPPED`).
   - Verifies sender/receiver balances change exactly once.
2. **`tamperedCiphertextIsRejected`**:
   - Flips a byte in the encrypted ciphertext.
   - Verifies that `BridgeIngestionService` catches the decryption failure and returns `INVALID`.
3. **`encryptDecryptRoundTrip`**:
   - Tests symmetric/asymmetric hybrid encryption and decryption roundtrip correctness.

---

## Inspecting Logs & Debugging

- **Log Pattern (`application.properties`)**:
  ```properties
  logging.level.com.demo.upimesh=INFO
  logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n
  ```
- **Console Output**:
  - Watch the terminal running `spring-boot:run` to see log messages for packet injection, gossip rounds, duplicate drops, and successful settlements.
- **H2 Console Inspection**:
  - Open `http://localhost:8080/h2-console` in your browser to inspect database tables (`accounts`, `transactions`) live while interacting with the dashboard.
