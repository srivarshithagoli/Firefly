# 03 — Project Structure

This document explains the directory layout and the responsibility of every important file in the codebase.

```text
upi-offline-mesh/
├── pom.xml                                  # Maven project configuration & dependencies
├── README.md                                # Project intro and usage instructions
├── mvnw, mvnw.cmd                           # Maven wrapper scripts (build without pre-installing Maven)
├── .mvn/                                    # Maven wrapper configuration and JAR
├── docs/                                    # Comprehensive documentation directory
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties       # Server port, H2 database URL, TTL configurations
    │   │   └── templates/
    │   │       └── dashboard.html           # Thymeleaf web dashboard UI
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java      # Spring Boot application main entry point
    │       │
    │       ├── config/
    │       │   └── AppConfig.java           # Enables Spring scheduled tasks (@EnableScheduling)
    │       │
    │       ├── crypto/
    │       │   ├── ServerKeyHolder.java     # Generates and holds the server's RSA-2048 keypair
    │       │   └── HybridCryptoService.java # RSA-OAEP + AES-256-GCM encryption, decryption & hashing
    │       │
    │       ├── model/
    │       │   ├── Account.java             # JPA Entity representing a bank account (VPA, balance, @Version)
    │       │   ├── AccountRepository.java   # Spring Data JPA interface for Account CRUD
    │       │   ├── Transaction.java         # JPA Entity representing the immutable ledger of settled/rejected txs
    │       │   ├── TransactionRepository.java # Spring Data JPA interface for Transaction queries
    │       │   ├── MeshPacket.java          # Wire format packet object (packetId, ttl, createdAt, ciphertext)
    │       │   └── PaymentInstruction.java  # Decrypted payload structure (senderVpa, receiverVpa, amount, pinHash, nonce, signedAt)
    │       │
    │       ├── service/
    │       │   ├── DemoService.java         # Seeds initial accounts and builds/encrypts test packets
    │       │   ├── VirtualDevice.java       # Represents a simulated phone in the mesh holding packets
    │       │   ├── MeshSimulatorService.java# Simulates peer-to-peer gossip broadcasts and bridge uploads
    │       │   ├── IdempotencyService.java  # ConcurrentHashMap-based deduplication cache (JVM local SETNX)
    │       │   ├── SettlementService.java   # @Transactional business logic executing debits, credits, and ledger logging
    │       │   └── BridgeIngestionService.java# Orchestrates the core pipeline: hash -> claim -> decrypt -> freshness -> settle
    │       │
    │       └── controller/
    │           ├── ApiController.java       # REST endpoints for server key, demo actions, mesh sim, and bridge ingest
    │           └── DashboardController.java # Serves the dashboard HTML at the root URL (/)
    │
    └── test/
        └── java/com/demo/upimesh/
            └── IdempotencyConcurrencyTest.java # Concurrency tests for idempotency, tampered packets, and roundtrips
```

---

## Detailed File Responsibilities

### Root Configuration Files
- **`pom.xml`**: Defines Maven build settings, parent `spring-boot-starter-parent` (v3.3.5), Java version (17), and all runtime/test dependencies (Web, Thymeleaf, Data JPA, H2, Validation, Jackson).
- **`application.properties`**: Configures server port (`8080`), H2 in-memory database connection (`jdbc:h2:mem:upimesh`), database console enablement, logging format, and TTL timeouts (86400 seconds for idempotency and packet max age).

### Core Package (`com.demo.upimesh`)
- **`UpiMeshApplication.java`**: Contains the `main` method and `@SpringBootApplication` annotation to bootstrap the Spring container.

### Config Package (`com.demo.upimesh.config`)
- **`AppConfig.java`**: Annotated with `@Configuration` and `@EnableScheduling` to enable background tasks (such as evicting expired items from `IdempotencyService`).

### Crypto Package (`com.demo.upimesh.crypto`)
- **`ServerKeyHolder.java`**: Uses `@PostConstruct` to generate an RSA 2048-bit keypair on startup. Exposes public key accessors and base64 encoders.
- **`HybridCryptoService.java`**: Implements hybrid encryption (`RSA-OAEP` wrapping an `AES-256-GCM` session key), decryption, and `SHA-256` ciphertext hashing.

### Model Package (`com.demo.upimesh.model`)
- **`Account.java`**: JPA entity storing virtual payment addresses (`vpa`), holder names, decimal balances, and an optimistic locking version field (`@Version`).
- **`AccountRepository.java`**: Standard Spring Data JPA repository for `Account`.
- **`Transaction.java`**: JPA entity recording settled and rejected transactions with a unique database index on `packetHash`.
- **`TransactionRepository.java`**: Spring Data JPA repository providing query methods like `findTop20ByOrderByIdDesc` and `existsByPacketHash`.
- **`MeshPacket.java`**: Data class representing packets transmitted over the mesh (outer readable fields + opaque base64 `ciphertext`).
- **`PaymentInstruction.java`**: Data class representing the decrypted payload containing sender, receiver, amount, PIN hash, nonce, and timestamp.

### Service Package (`com.demo.upimesh.service`)
- **`DemoService.java`**: Seeds demo accounts on startup and constructs/encrypts test packets mimicking sender mobile devices.
- **`VirtualDevice.java`**: Simulates a mobile device holding packets in a thread-safe `ConcurrentHashMap`.
- **`MeshSimulatorService.java`**: Manages a collection of `VirtualDevice` instances, handles packet injection, runs gossip rounds (TTL decrementing), and collects bridge uploads.
- **`IdempotencyService.java`**: Implements an in-memory deduplication cache (`ConcurrentHashMap.putIfAbsent`) with scheduled expiration.
- **`SettlementService.java`**: Executed within `@Transactional` boundaries to debit senders, credit receivers, and persist ledger records.
- **`BridgeIngestionService.java`**: Coordinates the multi-step server ingestion pipeline (hash, claim, decrypt, freshness check, settlement).

### Controller Package (`com.demo.upimesh.controller`)
- **`ApiController.java`**: REST API controller handling `/api/server-key`, `/api/demo/send`, `/api/mesh/*`, and `/api/bridge/ingest`.
- **`DashboardController.java`**: MVC controller mapping `GET /` to `dashboard.html`.

### Test Directory (`src/test/java/com/demo/upimesh`)
- **`IdempotencyConcurrencyTest.java`**: Contains Spring Boot integration tests verifying concurrent bridge uploads (idempotency), tampering detection, and cryptographic roundtrips.
