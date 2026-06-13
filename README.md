# PostgreSQL 19 Beta — Bidirectional Logical Replication POC

> **Proof of Concept**: Native bidirectional (multi-master) logical replication using PostgreSQL 19 Beta 1 with Docker — demonstrated on a banking database schema with `customers`, `accounts`, and `transactions` tables.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start — Clean Setup](#quick-start--clean-setup)
- [Step 1 — Start Containers](#step-1--start-containers)
- [Step 2 — Create Banking Schema](#step-2--create-banking-schema)
- [Step 3 — Grant Replication Role](#step-3--grant-replication-role)
- [Step 4 — Create Publications](#step-4--create-publications)
- [Step 5 — Create Subscriptions](#step-5--create-subscriptions)
- [Step 6 — Verify Replication is Active](#step-6--verify-replication-is-active)
- [Testing Bidirectional Replication](#testing-bidirectional-replication)
- [Monitoring](#monitoring)
- [Clean Teardown](#clean-teardown)
- [Key Concepts](#key-concepts)
- [Caveats & Production Considerations](#caveats--production-considerations)

---

## Overview

PostgreSQL **native bidirectional logical replication** was introduced in **PostgreSQL 16** via the `origin` parameter on subscriptions. Before this, two-way replication caused infinite loops — each node would reapply replicated data without knowing whether it originated locally or remotely.

The fix: `origin = none` instructs each subscriber to replicate **only locally written changes**, breaking the loop.

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   NODE 1 (pg_node1:5433)    NODE 2 (pg_node2:5434)     │
│   ┌─────────────────┐        ┌─────────────────┐       │
│   │   publisher:    │──────▶ │   subscriber:   │       │
│   │   pub_node1     │        │   sub_from_node1│       │
│   │                 │        │                 │       │
│   │   subscriber:   │◀────── │   publisher:    │       │
│   │   sub_from_node2│        │   pub_node2     │       │
│   └─────────────────┘        └─────────────────┘       │
│                                                         │
│         origin = none  ←── prevents infinite loops      │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works

| Component | Role |
|-----------|------|
| `pub_node1` | Node1 publishes its local writes |
| `pub_node2` | Node2 publishes its local writes |
| `sub_from_node2` on Node1 | Node1 subscribes to Node2's changes |
| `sub_from_node1` on Node2 | Node2 subscribes to Node1's changes |
| `origin = none` | Each subscriber only replicates **locally originated** rows — preventing echo/loop |
| UUID primary keys | `gen_random_uuid()` ensures no primary key collisions across nodes |

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/) installed
- Internet access to pull `postgres:19beta1` from Docker Hub

> ⚠️ **PostgreSQL 19 is Beta software. Do not use in production.**

---

## Project Structure

```
pg-bidir-replica/
└── docker-compose.yml
```

### `docker-compose.yml`

```yaml
version: '3.9'

services:
  node1:
    image: postgres:19beta1
    container_name: pg_node1
    environment:
      POSTGRES_USER: pguser
      POSTGRES_PASSWORD: pgpass
      POSTGRES_DB: bankdb
    ports:
      - "5433:5432"
    command: >
      postgres
        -c wal_level=logical
        -c max_replication_slots=10
        -c max_wal_senders=10
    networks:
      - pgnet

  node2:
    image: postgres:19beta1
    container_name: pg_node2
    environment:
      POSTGRES_USER: pguser
      POSTGRES_PASSWORD: pgpass
      POSTGRES_DB: bankdb
    ports:
      - "5434:5432"
    command: >
      postgres
        -c wal_level=logical
        -c max_replication_slots=10
        -c max_wal_senders=10
    networks:
      - pgnet

networks:
  pgnet:
    driver: bridge
```

---

## Quick Start — Clean Setup

If you have leftover containers from a previous run, always clean up first:

```bash
docker compose down -v
docker rm -f pg_node1 pg_node2 2>/dev/null
docker volume prune -f
```

---

## Step 1 — Start Containers

```bash
docker compose up -d
sleep 3
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE              PORTS                     NAMES
xxxxxxxxxxxx   postgres:19beta1   0.0.0.0:5433->5432/tcp   pg_node1
xxxxxxxxxxxx   postgres:19beta1   0.0.0.0:5434->5432/tcp   pg_node2
```

---

## Step 2 — Create Banking Schema

Run on **both nodes**. Tables must exist on both sides before subscriptions are created. We use `UUID` primary keys to avoid sequence collision across nodes.

```bash
# Node 1
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE customers (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name  TEXT NOT NULL,
  email      TEXT UNIQUE NOT NULL,
  phone      TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE accounts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  account_no  TEXT UNIQUE NOT NULL,
  balance     NUMERIC(15,2) DEFAULT 0.00,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE transactions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id  UUID NOT NULL REFERENCES accounts(id),
  txn_type    TEXT NOT NULL CHECK (txn_type IN ('CREDIT','DEBIT')),
  amount      NUMERIC(15,2) NOT NULL,
  description TEXT,
  txn_at      TIMESTAMPTZ DEFAULT now()
);
"

# Node 2
docker exec -it pg_node2 psql -U pguser -d bankdb -c "
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE customers (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name  TEXT NOT NULL,
  email      TEXT UNIQUE NOT NULL,
  phone      TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE accounts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  account_no  TEXT UNIQUE NOT NULL,
  balance     NUMERIC(15,2) DEFAULT 0.00,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE transactions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id  UUID NOT NULL REFERENCES accounts(id),
  txn_type    TEXT NOT NULL CHECK (txn_type IN ('CREDIT','DEBIT')),
  amount      NUMERIC(15,2) NOT NULL,
  description TEXT,
  txn_at      TIMESTAMPTZ DEFAULT now()
);
"
```

---

## Step 3 — Grant Replication Role

```bash
docker exec -it pg_node1 psql -U pguser -d bankdb -c "ALTER ROLE pguser REPLICATION LOGIN;"
docker exec -it pg_node2 psql -U pguser -d bankdb -c "ALTER ROLE pguser REPLICATION LOGIN;"
```

---

## Step 4 — Create Publications

Each node publishes all three tables:

```bash
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
CREATE PUBLICATION pub_node1 FOR TABLE customers, accounts, transactions;
"

docker exec -it pg_node2 psql -U pguser -d bankdb -c "
CREATE PUBLICATION pub_node2 FOR TABLE customers, accounts, transactions;
"
```

---

## Step 5 — Create Subscriptions

> **Critical**: `origin = none` prevents infinite replication loops.
> **Critical**: `copy_data = false` because tables are empty — no initial sync needed.

```bash
# Node1 subscribes to Node2
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
CREATE SUBSCRIPTION sub_from_node2
  CONNECTION 'host=pg_node2 port=5432 user=pguser password=pgpass dbname=bankdb'
  PUBLICATION pub_node2
  WITH (origin = none, copy_data = false);
"

# Node2 subscribes to Node1
docker exec -it pg_node2 psql -U pguser -d bankdb -c "
CREATE SUBSCRIPTION sub_from_node1
  CONNECTION 'host=pg_node1 port=5432 user=pguser password=pgpass dbname=bankdb'
  PUBLICATION pub_node1
  WITH (origin = none, copy_data = false);
"
```

Expected output for each:
```
NOTICE:  created replication slot "sub_from_nodeX" on publisher
CREATE SUBSCRIPTION
```

---

## Step 6 — Verify Replication is Active

```bash
echo "=== Node1 subscriptions ==="
docker exec -it pg_node1 psql -U pguser -d bankdb -c \
  "SELECT subname, subenabled FROM pg_subscription;"

echo "=== Node2 subscriptions ==="
docker exec -it pg_node2 psql -U pguser -d bankdb -c \
  "SELECT subname, subenabled FROM pg_subscription;"

echo "=== Node1 replication slots ==="
docker exec -it pg_node1 psql -U pguser -d bankdb -c \
  "SELECT slot_name, active FROM pg_replication_slots;"

echo "=== Node2 replication slots ==="
docker exec -it pg_node2 psql -U pguser -d bankdb -c \
  "SELECT slot_name, active FROM pg_replication_slots;"
```

Expected — all subscriptions `subenabled = t`, all slots `active = t`.

---

## Testing Bidirectional Replication

### TEST 1 — Insert customer on Node1, verify on Node2

```bash
echo "--- [NODE1] Register customer: Clement ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
INSERT INTO customers (full_name, email, phone)
VALUES ('Clement', 'clement@bank.com', '+91-9000000001');
"

sleep 1

echo "--- [NODE2] Should see Clement ---"
docker exec -it pg_node2 psql -U pguser -d bankdb -c \
  "SELECT full_name, email FROM customers ORDER BY full_name;"
```

---

### TEST 2 — Insert customer on Node2, verify on Node1

```bash
echo "--- [NODE2] Register customer: Colin ---"
docker exec -it pg_node2 psql -U pguser -d bankdb -c "
INSERT INTO customers (full_name, email, phone)
VALUES ('Colin', 'colin@bank.com', '+91-9000000002');
"

sleep 1

echo "--- [NODE1] Should see Clement + Colin ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c \
  "SELECT full_name, email FROM customers ORDER BY full_name;"
```

---

### TEST 3 — Simultaneous inserts on both nodes

```bash
echo "--- [NODE1 + NODE2] Register Shruti and Sindhu simultaneously ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
INSERT INTO customers (full_name, email, phone)
VALUES ('Shruti', 'shruti@bank.com', '+91-9000000003');
" &

docker exec -it pg_node2 psql -U pguser -d bankdb -c "
INSERT INTO customers (full_name, email, phone)
VALUES ('Sindhu', 'sindhu@bank.com', '+91-9000000004');
" &

wait
sleep 2

echo "--- [NODE1] Final: should show all 4 customers: Clement, Colin, Shruti, Sindhu ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c \
  "SELECT full_name, email FROM customers ORDER BY full_name;"

echo "--- [NODE2] Final: should show all 4 customers: Clement, Colin, Shruti, Sindhu ---"
docker exec -it pg_node2 psql -U pguser -d bankdb -c \
  "SELECT full_name, email FROM customers ORDER BY full_name;"
```

Expected on **both nodes**:
```
 full_name | email
-----------+---------------------
 Clement   | clement@bank.com
 Colin     | colin@bank.com
 Shruti    | shruti@bank.com
 Sindhu    | sindhu@bank.com
```

---

### TEST 4 — Account + Transaction across nodes

```bash
# Create account for Clement on Node1
echo "--- [NODE1] Open account for Clement ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
INSERT INTO accounts (customer_id, account_no, balance)
SELECT id, 'ACC-CLEMENT-001', 50000.00
FROM customers WHERE email = 'clement@bank.com';
"

sleep 1

# Credit transaction entered on Node2 (simulating a branch in another city)
echo "--- [NODE2] Credit transaction for Clement's account (from Node2 branch) ---"
docker exec -it pg_node2 psql -U pguser -d bankdb -c "
INSERT INTO transactions (account_id, txn_type, amount, description)
SELECT id, 'CREDIT', 10000.00, 'Salary credit - June 2026'
FROM accounts WHERE account_no = 'ACC-CLEMENT-001';
"

sleep 1

# Verify full ledger visible on Node1
echo "--- [NODE1] Full ledger view ---"
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
SELECT
  c.full_name,
  a.account_no,
  a.balance,
  t.txn_type,
  t.amount,
  t.description,
  t.txn_at
FROM transactions t
JOIN accounts a ON t.account_id = a.id
JOIN customers c ON a.customer_id = c.id
ORDER BY t.txn_at;
"
```

---

## Monitoring

```bash
# Subscription worker status
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
SELECT subname, pid, received_lsn, last_msg_send_time, last_msg_receipt_time
FROM pg_stat_subscription;
"

# WAL sender status (what Node1 is sending to Node2)
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
SELECT pid, usename, application_name, state, sent_lsn, write_lsn
FROM pg_stat_replication;
"

# Replication lag
docker exec -it pg_node1 psql -U pguser -d bankdb -c "
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
"
```

---

## Clean Teardown

```bash
docker compose down -v
docker rm -f pg_node1 pg_node2 2>/dev/null
docker volume prune -f
```

---

## Key Concepts

| Concept | Detail |
|---------|--------|
| `wal_level = logical` | Required to enable logical decoding of WAL |
| `origin = none` | Only replicate locally-originated writes — **the key to preventing loops** |
| `copy_data = false` | Skip initial table sync (use `true` only when seeding from existing data into an empty subscriber) |
| UUID primary keys | `gen_random_uuid()` guarantees no PK collision across nodes writing independently |
| Pub/Sub model | Each node is both publisher and subscriber simultaneously |

---

## Caveats & Production Considerations

| Topic | Notes |
|-------|-------|
| **Conflict resolution** | PostgreSQL native BDR has no automatic conflict resolution. Concurrent writes to the **same row** from both nodes may result in last-write-wins data loss. Use application-level partitioning (e.g., each region owns certain rows) to avoid this. |
| **Foreign key ordering** | When replicating across nodes, parent rows must arrive before child rows. In practice this works because replication is ordered per-transaction, but be aware during bulk loads. |
| **DDL not replicated** | Schema changes (`ALTER TABLE`, `CREATE INDEX`, etc.) are **not** replicated. You must run DDL on each node manually. |
| **Sequences** | `SERIAL` / `SEQUENCE` are not replicated and will collide. Always use `gen_random_uuid()` or application-managed unique keys for BDR setups. |
| **Beta disclaimer** | PostgreSQL 19 Beta 1 (released June 4, 2026) is for testing only. Not for production use. |
| **This feature since PG16** | `origin = none` was introduced in PostgreSQL 16. This POC uses PG19 Beta but the same steps apply to PG16, PG17, and PG18. |

---

## References

- [PostgreSQL 19 Beta 1 Release Announcement](https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/)
- [PostgreSQL Logical Replication Docs](https://www.postgresql.org/docs/current/logical-replication.html)
- [CREATE SUBSCRIPTION — origin parameter](https://www.postgresql.org/docs/current/sql-createsubscription.html)
- [postgres Docker Hub image](https://hub.docker.com/_/postgres)

---

## Author

**Clement** — PostgreSQL Bidirectional Replication POC  
Tested on: PostgreSQL 19 Beta 1 · Docker · Ubuntu (WSL2)
