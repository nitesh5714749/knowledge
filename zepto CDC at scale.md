# Zepto Debezium at Scale — Key Concepts Cheat Sheet

> A plain-English reference so you can re-read the blog easily next time.

---

## 1. What is CDC (Change Data Capture)?

**What it is:** A technique to capture every INSERT / UPDATE / DELETE that happens in a source database and stream those changes elsewhere in real time.

**Why Zepto used it:** They needed to sync order/inventory state changes from their transactional DB to downstream systems (analytics, search, notifications) without polling the DB constantly.

**Tool used:** **Debezium** — an open-source CDC framework that reads Postgres's WAL (Write Ahead Log) and publishes changes to Kafka.

---

## 2. WAL (Write Ahead Log)

**What it is:** Postgres writes every change to a log file *before* applying it to the actual table. This log is called the WAL.

**Why it matters for CDC:** Debezium reads the WAL like a "change feed" — it sees every row-level change in order, without touching the actual tables. Zero impact on the source DB's query performance.

---

## 3. Upsert (INSERT … ON CONFLICT DO UPDATE)

**What it is:** A single SQL statement that either inserts a new row OR updates an existing one, depending on whether the primary key already exists.

```sql
INSERT INTO orders (id, state) VALUES (123, 'STATE_D')
ON CONFLICT (id) DO UPDATE SET state = EXCLUDED.state;
```

**Why CDC uses it:** When syncing changes to a destination DB, you don't know if the row exists yet — upsert handles both cases automatically.

---

## 4. reWriteBatchedInserts=true

**What it is:** A JDBC driver flag that merges a batch of individual INSERT statements into one single multi-row INSERT for performance.

**The problem it caused:** If the same primary key appears more than once in a batch (e.g., same order updated twice), Postgres throws:
```
ERROR: ON CONFLICT DO UPDATE command cannot affect row a second time.
```
Because Postgres cannot resolve which value wins within a single statement.

**Root cause:** CDC pipelines naturally produce multiple events for the same row (same order going STATE_A → B → C → D in milliseconds), which collide when merged.

---

## 5. Heap & Index Writes

**Heap:** Where the actual table rows live on disk. Every UPDATE writes a *new copy* of the row (Postgres never overwrites in place).

**Index:** A separate lookup structure that points to where each row lives. When a row moves (due to an update), the index pointer must be updated too.

**Write Amplification:** One logical change (e.g., state update) causes multiple physical disk writes:
- 1 heap write (new row copy)
- N index writes (one per index on the table)
- 1 dead row mark (old version)

---

## 6. HOT Updates (Heap Only Tuple)

**What it is:** A Postgres optimization that skips updating indexes when:
1. The updated column has NO index on it, AND
2. The new row version fits on the **same heap page**

**Why it breaks in CDC:** The `state` column is indexed (you query by state constantly). Every CDC event changes `state` → HOT cannot apply → every update rewrites all indexes → massive write amplification at scale.

---

## 7. Index Write Amplification (The Breaking Point)

**The scenario:** One order goes STATE_A → B → C → D in milliseconds. Without any optimization, Postgres processes all 4 separately:

```
Each state change = 1 heap write + N index writes
4 state changes   = 4 heap writes + 4×N index writes
```

At Zepto's scale (millions of orders), this overwhelmed the Postgres writer instance during peak traffic.

---

## 8. The Reduction Buffer (The Fix)

**What it is:** An in-memory deduplication layer that sits between Debezium (Kafka consumer) and Postgres. Before flushing a batch to the DB, it collapses all events per primary key down to just the **latest one**.

**How it works:**
```
Batch arrives:
  id=123, STATE_A  ← discard
  id=123, STATE_B  ← discard
  id=123, STATE_C  ← discard
  id=123, STATE_D  ← KEEP (latest)
  id=456, STATE_A  ← KEEP (only one)

Send to Postgres:
  id=123, STATE_D
  id=456, STATE_A
```

**Why it's powerful:**
- Eliminates duplicate keys → `reWriteBatchedInserts` now works safely
- Reduces DB writes by 50-80% during peak traffic
- Only the final state matters — intermediate states are never needed

---

## 9. AAS (Average Active Sessions)

**What it is:** A metric showing how many database sessions are actively working or waiting at any given moment.

**Healthy vs unhealthy:**
```
AAS = 1-5    → calm, queries completing quickly       ✅
AAS = 50+    → sessions piling up, DB overwhelmed     ❌
```

**What Zepto saw:** During peak traffic, AAS for COMMIT operations spiked massively — hundreds of write sessions queuing up because disk couldn't flush fast enough. This created a death spiral: more waiting → slower flushes → even more waiting.

---

## 10. COMMIT & WAL Flushing

**What COMMIT does:** Makes a transaction permanent by flushing changes to disk (WAL). Until COMMIT, nothing is saved.

**Why it's expensive at scale:** Every batch flush ends with a COMMIT. During peak traffic, hundreds of COMMITs hit simultaneously. Disk I/O becomes the bottleneck — sessions queue up waiting for their COMMIT to complete, spiking AAS.

---

## How Everything Connects

```
Debezium reads WAL
      ↓
Kafka receives change events
      ↓
JDBC Sink Consumer picks up batch
      ↓
[REDUCTION BUFFER]  ← deduplicates by primary key, keeps latest only
      ↓
Deduplicated batch → reWriteBatchedInserts merges into 1 SQL statement
      ↓
Postgres: 1 heap write + 1 index write per unique row
      ↓
COMMIT: fast, AAS stays low
```

---

## Quick Reference Table

| Concept | One-line summary |
|---|---|
| CDC | Stream every DB change to other systems in real time |
| WAL | Postgres change log that Debezium reads |
| Upsert | Insert if new, update if exists — one statement |
| reWriteBatchedInserts | Merges batch into one SQL — breaks on duplicate keys |
| Heap write | Physical write of new row copy on disk |
| Index write | Updating lookup pointer when a row moves |
| HOT update | Skips index writes if indexed column unchanged (often blocked in CDC) |
| Write amplification | 1 logical change → many physical disk writes |
| Reduction Buffer | In-memory dedup — keep only latest event per primary key |
| AAS | How many DB sessions are active/waiting right now |
| COMMIT spike | Too many flushes at once → sessions queue → DB slows down |
