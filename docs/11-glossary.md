# 11 — Glossary

This document defines unfamiliar technical terms in simple English, with examples relevant to this project.

---

- **AES-256-GCM (Advanced Encryption Standard with Galois/Counter Mode)**: A fast symmetric encryption algorithm that both encrypts data and provides tamper-proofing (authentication tags). Used here to encrypt payment payloads.
- **Asymmetric Encryption**: A cryptographic system using a public key (to encrypt) and a private key (to decrypt). Used here via RSA so sender phones can encrypt payloads while only the backend server can decrypt them.
- **Bridge Node**: A device in the mesh network that possesses internet connectivity (e.g., 4G/Wi-Fi) and uploads accumulated offline payment packets to the backend server.
- **Ciphertext**: Encrypted, unreadable data resulting from applying an encryption algorithm to plaintext.
- **Dependency Injection (DI)**: A design pattern where Spring automatically supplies (injects) dependent objects (`@Autowired`) rather than classes creating them manually.
- **Gossip Protocol**: A peer-to-peer communication method where devices periodically exchange data packets with randomly chosen neighbors until all devices receive the data.
- **Idempotency**: The property where an operation can be applied multiple times without changing the result beyond the initial application. Used here to ensure duplicate packet deliveries settle exactly once.
- **JPA (Java Persistence API) / Hibernate**: A framework that maps Java database entity classes to relational database tables and handles SQL operations automatically.
- **Nonce (Number used ONCE)**: A unique random value (UUID) included in every payment instruction to ensure that even identical payments produce unique ciphertexts.
- **Optimistic Locking**: A concurrency control strategy using a version number (`@Version`) to detect and prevent conflicting concurrent database updates.
- **RSA-OAEP**: An asymmetric encryption scheme using RSA padding to safely encrypt small data (such as AES session keys).
- **TTL (Time-To-Live)**: A counter inside a packet (`MeshPacket.ttl`) representing how many relay hops remaining before the packet stops propagating.
- **VPA (Virtual Payment Address)**: A unique identifier representing a user's bank account address, e.g., `alice@demo`.
