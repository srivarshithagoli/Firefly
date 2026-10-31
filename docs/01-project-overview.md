# 01 — Project Overview

## What the Application Does
**UPI Offline Mesh** is a Spring Boot application designed to demonstrate how peer-to-peer (P2P) mesh networking (such as Bluetooth-style packet relaying) can facilitate offline financial transactions (like UPI payments) when devices have zero internet connectivity. 

Imagine you are in a basement with no cellular signal or Wi-Fi. You want to pay a friend ₹500. Your phone encrypts your payment instruction and broadcasts it wirelessly to nearby phones. Those phones (strangers' devices) act as relays ("hops"), forwarding the packet from device to device until eventually, *someone* walks out of the basement into an area with 4G internet. That device (the "bridge node") automatically uploads the accumulated payment packets to the backend server. The server decrypts them, prevents duplicate processing, and settles the funds between bank accounts.

## Main Features
1. **Hybrid Cryptographic Envelope (`HybridCryptoService.java`)**: Protects payment instructions end-to-end so untrusted relay devices cannot read or modify transaction details.
2. **Mesh Network Simulator (`MeshSimulatorService.java`, `VirtualDevice.java`)**: Simulates virtual smartphones, packet injection, TTL (Time-To-Live) decrementing, and multi-hop gossip broadcasts entirely in-memory.
3. **Atomic Idempotency Engine (`IdempotencyService.java`)**: Prevents duplicate settlements when multiple bridge nodes upload the exact same packet simultaneously.
4. **Optimistic Locking & ACID Database Transactions (`SettlementService.java`)**: Safely updates sender and receiver balances in an embedded H2 database while maintaining an immutable transaction ledger (`Transaction.java`).
5. **Interactive Dark-Mode Dashboard (`dashboard.html`, `ApiController.java`)**: Provides a graphical UI to compose payments, run gossip rounds, flush bridge uploads, reset state, and monitor account balances and transaction histories in real time.

## System Boundaries
- **What is inside the repository**: The server-side Spring Boot backend, cryptographic encryption/decryption routines, in-memory mesh simulator, database models, REST controllers, Thymeleaf dashboard UI, and concurrency/idempotency unit tests.
- **What is outside the repository**: Actual mobile client apps (Android/iOS apps written in Kotlin/Swift), physical Bluetooth/Wi-Fi Direct hardware communication stacks, real banking core integration (such as NPCI / RBI payment rails), and distributed key management systems (HSM/KMS).

## Implemented vs. Simulated vs. Missing

| Feature / Aspect | Implementation Status | Details & Source Reference |
|---|---|---|
| **Server-side Decryption & Verification** | Fully Implemented | `HybridCryptoService.java`, `BridgeIngestionService.java` |
| **Idempotency & Concurrency Control** | Fully Implemented | `IdempotencyService.java`, `IdempotencyConcurrencyTest.java` |
| **Ledger Settlement & H2 Database** | Fully Implemented | `SettlementService.java`, `Account.java`, `Transaction.java` |
| **Mesh Gossip & Virtual Devices** | Simulated in Memory | `MeshSimulatorService.java`, `VirtualDevice.java` |
| **Sender Phone App & Packet Creation** | Simulated on Server | `DemoService.java` generates and encrypts test packets |
| **Physical Bluetooth BLE / Wi-Fi Direct** | Missing (Simulated) | No native mobile Bluetooth stack or GATT server/client code exists |
| **Real HSM / KMS Key Management** | Simulated in Memory | `ServerKeyHolder.java` generates a fresh RSA-2048 keypair on startup |
| **Real Bank Core / UPI Gateway** | Simulated (Local DB) | Accounts and balances are stored locally in H2 database tables |
