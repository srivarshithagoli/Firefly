# 06 — Request and Data Flow

This document traces the three most important request and execution flows end-to-end through the codebase, citing exact file paths, class names, and method names.

---

## Flow 1: Composing and Injecting a Payment (`POST /api/demo/send`)

When a user clicks **"📤 Inject into Mesh"** on the dashboard, the following steps occur:

1. **HTTP Request (`dashboard.html` → `ApiController.java`)**:
   - JavaScript sends a `POST` request to `/api/demo/send` with JSON body containing `senderVpa`, `receiverVpa`, `amount`, `pin`, `ttl`, and `startDevice`.
   - `ApiController.demoSend(@RequestBody DemoSendRequest req)` handles the endpoint.

2. **Packet Creation & Encryption (`DemoService.java`)**:
   - `ApiController` calls `demo.createPacket(...)`.
   - `DemoService.createPacket()` builds a `PaymentInstruction` containing:
     - `senderVpa`, `receiverVpa`, `amount`
     - `pinHash` (SHA-256 hash of the PIN)
     - `nonce` (UUID generated via `UUID.randomUUID().toString()`)
     - `signedAt` (`Instant.now().toEpochMilli()`)
   - It calls `crypto.encrypt(instruction, serverKey.getPublicKey())` (`HybridCryptoService.java`):
     - Generates a one-time AES-256 key.
     - Encrypts the JSON payment instruction using `AES/GCM/NoPadding`.
     - Encrypts the AES session key using `RSA/ECB/OAEPWithSHA-256AndMGF1Padding` with the server's public key (`ServerKeyHolder`).
     - Concatenates `[RSA-encrypted AES key][12-byte IV][AES ciphertext + tag]` and encodes to Base64.
   - Wraps the ciphertext in a `MeshPacket` object (`packetId`, `ttl`, `createdAt`, `ciphertext`).

3. **Mesh Injection (`MeshSimulatorService.java`, `VirtualDevice.java`)**:
   - `ApiController` calls `mesh.inject(startDevice, packet)`.
   - `MeshSimulatorService` looks up `VirtualDevice` (`phone-alice`) and calls `sender.hold(packet)`, storing the packet in its thread-safe `ConcurrentHashMap` of held packets.

4. **Response**:
   - `ApiController` returns a JSON response containing `packetId`, `ciphertextPreview`, `ttl`, and `injectedAt`.

---

## Flow 2: Running Gossip Rounds (`POST /api/mesh/gossip`)

When a user clicks **"🔄 Run Gossip Round"**:

1. **HTTP Request (`dashboard.html` → `ApiController.java`)**:
   - `POST /api/mesh/gossip` invokes `ApiController.meshGossip()`.

2. **Gossip Execution (`MeshSimulatorService.java`)**:
   - Calls `mesh.gossipOnce()`.
   - Takes a snapshot of what every `VirtualDevice` holds.
   - Iterates through all devices and packets. For each packet with `ttl > 0`:
     - Copies the packet to every other device that doesn't already hold it.
     - Decrements TTL (`copy.setTtl(pkt.getTtl() - 1)`).
   - Returns a `GossipResult` with total transfers and updated device packet counts.

---

## Flow 3: Bridge Node Ingestion & Settlement (`POST /api/bridge/ingest`)

When bridge nodes upload packets (either via **"📡 Bridges Upload to Backend"** or directly to the production endpoint):

1. **HTTP Request (`ApiController.java`)**:
   - `POST /api/bridge/ingest` receives a `MeshPacket` in the request body, along with request headers `X-Bridge-Node-Id` and `X-Hop-Count`.
   - Calls `bridge.ingest(packet, bridgeNodeId, hopCount)` (`BridgeIngestionService.java`).

2. **Pipeline Execution (`BridgeIngestionService.java`)**:
   - **Step A: Ciphertext Hashing**:
     - `String packetHash = crypto.hashCiphertext(packet.getCiphertext());` (`HybridCryptoService.java` computes SHA-256 hex).
   - **Step B: Idempotency Gate**:
     - `if (!idempotency.claim(packetHash))` (`IdempotencyService.java`):
       - Attempts `seen.putIfAbsent(packetHash, now)`. If an entry already exists, returns `false` (duplicate detected), returning `IngestResult.duplicate(packetHash)` immediately.
   - **Step C: Decryption**:
     - `instruction = crypto.decrypt(packet.getCiphertext());` (`HybridCryptoService.java`):
       - Unpacks RSA-encrypted AES key, IV, and ciphertext.
       - Decrypts AES key using server's private key (`ServerKeyHolder`).
       - Decrypts and verifies GCM authentication tag. If tampered, throws an exception caught and returned as `IngestResult.invalid(packetHash, "decryption_failed")`.
   - **Step D: Freshness Check (Replay Protection)**:
     - Calculates `ageSeconds`. If `ageSeconds > maxAgeSeconds` (86,400s / 24 hours), rejects as `stale_packet`.
   - **Step E: Settlement (`SettlementService.java`)**:
     - Wrapped in `@Transactional`.
     - Fetches sender and receiver `Account` entities from `AccountRepository`.
     - Verifies sender balance is sufficient (`sender.getBalance().compareTo(amount) < 0` records rejected transaction).
     - Subtracts amount from sender balance, adds to receiver balance, and saves accounts.
     - Persists a new `Transaction` entity in `TransactionRepository` with status `SETTLED` and unique `packetHash`.

3. **Response**:
   - Returns `IngestResult` (SETTLED, DUPLICATE_DROPPED, or INVALID) back to `ApiController` and client.
