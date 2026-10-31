# 07 — Database Guide

This document explains the database entities, tables, relationships, schema generation, persistence behavior, and H2 configuration based on source inspection.

---

## Configuration & H2 Setup

- **Database Engine**: H2 In-Memory Database (`org.h2.Driver`).
- **Connection URL**: `jdbc:h2:mem:upimesh` (configured in `application.properties`).
- **Schema Generation**: `spring.jpa.hibernate.ddl-auto=create-drop` (tables are created on startup and dropped on shutdown).
- **H2 Console**: Enabled at `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:upimesh`, Username: `sa`, Password: empty).

---

## Entities & Table Schemas

### 1. `Account` Entity (`Account.java`)
Represents a user's bank account and balance, identified by their Virtual Payment Address (VPA).
- **Table Name**: `accounts`
- **Columns**:
  - `vpa` (PK, String): Virtual Payment Address, e.g., `alice@demo`
  - `holder_name` (String, Not Null): User's display name
  - `balance` (BigDecimal, Not Null, precision=19, scale=2): Current account balance
  - `version` (Long): Optimistic locking version field (`@Version`)

### 2. `Transaction` Entity (`Transaction.java`)
Records every settled or rejected payment transaction processed by the backend.
- **Table Name**: `transactions`
- **Columns**:
  - `id` (PK, Long, GeneratedType.IDENTITY): Primary key
  - `packet_hash` (String, Not Null, Unique, length=64): SHA-256 hex of the encrypted packet (idempotency key & defense-in-depth index)
  - `sender_vpa` (String, Not Null): Sender's VPA
  - `receiver_vpa` (String, Not Null): Receiver's VPA
  - `amount` (BigDecimal, Not Null, precision=19, scale=2): Transaction amount
  - `signed_at` (Instant, Not Null): Timestamp when sender signed the packet offline
  - `settled_at` (Instant, Not Null): Timestamp when backend processed the packet
  - `bridge_node_id` (String, Not Null): Identifier of the bridge node delivering the packet
  - `hop_count` (int, Not Null): Number of relay hops the packet traversed
  - `status` (Enum: `SETTLED`, `REJECTED`, Not Null): Final transaction status

---

## Repositories

- **`AccountRepository.java`**: Extends `JpaRepository<Account, String>`. Provides CRUD operations for accounts (e.g., `findById`, `save`).
- **`TransactionRepository.java`**: Extends `JpaRepository<Transaction, Long>`. Provides custom query methods:
  - `findTop20ByOrderByIdDesc()`: Fetches the 20 most recent transactions for the dashboard.
  - `existsByPacketHash(String packetHash)`: Checks if a packet hash already exists in the ledger.

---

## Persistence Behavior & Concurrency Protections

1. **ACID Transactions (`SettlementService.java`)**:
   - The `settle()` method is annotated with `@Transactional`. This ensures that updating the sender balance, receiver balance, and inserting the transaction ledger record all happen atomically. If any step fails, the entire database transaction rolls back.
2. **Optimistic Locking (`Account.java`)**:
   - The `Account` entity features a `@Version Long version` field. Hibernate automatically checks version numbers during updates to prevent lost updates if concurrent transactions attempt to modify the same account balance.
3. **Database-Level Unique Index (`Transaction.java`)**:
   - The `packetHash` column has a unique index (`@Index(name = "idx_packet_hash", columnList = "packetHash", unique = true)`). This serves as a defense-in-depth fallback; even if the in-memory idempotency cache fails, the database rejects duplicate packet insertions.

---

## Example Supported Queries (H2 Console)

```sql
-- View all accounts and balances
SELECT vpa, holder_name, balance FROM accounts;

-- View recent transactions
SELECT id, packet_hash, sender_vpa, receiver_vpa, amount, status, settled_at FROM transactions ORDER BY settled_at DESC;

-- Check for specific packet settlement
SELECT * FROM transactions WHERE packet_hash = 'your_packet_hash_here';
```
