# Apache Iceberg Learning Lab

Hands-on Apache Iceberg learning using Spark SQL, an Iceberg REST Catalog, and RustFS (S3-compatible object storage).

> **Checkpoint:** Paused after schema evolution (Step 9). Continue with Step 10 (insert into evolved schema). The two-day accelerated bootcamp is approximately 35% complete by practical topic coverage, not an exam score.

## Learning objectives

- Understand the **catalog → metadata JSON → snapshot → manifest list → manifest → Parquet** chain.
- Practice snapshot creation, time travel, and rollback on an isolated table.
- Evolve schemas safely, then study partition evolution, UPDATE / DELETE / MERGE, maintenance, performance, and operational reliability.
- Finish with a tested fintech data-engineering capstone.

## Setup

This repository's working Codespaces environment uses Docker Compose with three services:

| Service | Purpose |
| --- | --- |
| `spark-iceberg` | Spark SQL and Iceberg integration |
| `iceberg-rest` | REST Catalog, backed by SQLite |
| `object-store` | RustFS S3-compatible warehouse |

**Note:** The current local Codespaces `docker-compose.yml` and project files must be reviewed and committed separately. They are not yet in the remote repository. The local REST catalog is persisted under `.local/iceberg-catalog/`; the existing RustFS data is bound to a preserved Docker volume. Neither belongs in Git.

After the local Compose file is committed, from Codespaces:

```bash
docker compose ps
docker exec spark-iceberg spark-sql -e "SELECT COUNT(*) FROM demo.training.transactions"
docker exec spark-iceberg spark-sql -e "SELECT COUNT(*) FROM demo.bootcamp.snapshot_lab"
```

Expected at this checkpoint: **8** rows in `demo.training.transactions` and **3** rows in `demo.bootcamp.snapshot_lab`.

## Verified experiments

| Topic | Result |
| --- | --- |
| Table creation and append | Confirmed |
| Iceberg metadata JSON and manifest traversal | Confirmed with boto3 + fastavro |
| Physical Parquet row locations | Confirmed using `input_file_name()` |
| REST catalog registration recovery | Recovered original `transactions` table via `register_table` |
| Snapshot lineage | Confirmed parent/child IDs from metadata JSON |
| Time travel | Historical query returned 3 rows while current table had 5 |
| Rollback | Current snapshot restored to initial 3-row version |
| Schema evolution | Added `payment_method` and renamed `status` → `transaction_status`; existing values preserved |

### Saved checkpoint

- **Recovered reference table:** `demo.training.transactions` — 8 records, left untouched during bootcamp.
- **Disposable lab table:** `demo.bootcamp.snapshot_lab` — 3 records after rollback.
- **Lab current snapshot:** `6004915092171797157` (initial 3 rows).
- **Lab former five-row snapshot:** `4146036655147643651` (historical; availability subject to retention).
- **Schema after Step 9:** `transaction_id BIGINT`, `customer_id BIGINT`, `amount DECIMAL(12,2)`, `transaction_status STRING`, `payment_method STRING`.
- Three existing records have `payment_method = NULL`.
- **Not yet run:** Step 10 INSERT of transaction `2006` with `payment_method='PAYNOW'`.

## Resume from here: Step 10

First verify the current state so the INSERT is not accidentally repeated:

```bash
docker compose ps
docker exec spark-iceberg spark-sql -e "SELECT transaction_id, transaction_status, payment_method FROM demo.bootcamp.snapshot_lab ORDER BY transaction_id"
```

If the table contains exactly IDs `2001`, `2002`, and `2003`, insert the next record **once**:

```bash
docker exec spark-iceberg spark-sql -e "
INSERT INTO demo.bootcamp.snapshot_lab
  (transaction_id, customer_id, amount, transaction_status, payment_method)
VALUES (2006, 606, 320.50, 'SUCCESS', 'PAYNOW');

SELECT transaction_id, transaction_status, payment_method
FROM demo.bootcamp.snapshot_lab ORDER BY transaction_id;
"
```

Expected: four rows; `payment_method` remains `NULL` for the three original rows, and `PAYNOW` for `2006`.

Then continue with partition evolution, UPDATE / DELETE / MERGE, performance, table maintenance, production-readiness tests, and the fintech capstone.

## Important reliability notes

- The local Docker environment previously encountered `RWLayer ... unexpectedly nil` after Codespaces restarts; root cause **not yet fixed**.
- Some Iceberg metadata-table SQL queries (for example `.files` / `.snapshots`) return REST `400 Suspicious Path Character`; the direct RustFS metadata inspection is the confirmed workaround.
- SQLite catalog metadata must persist outside ephemeral containers. The RustFS warehouse uses an existing Docker volume. **Git is not a backup of either datastore.**
- Do not commit `.local/`, `backups/`, virtual environments, credentials, logs, SQLite databases, or volume data. Inspect `git status` and staged diffs before publishing.

## Study approach

Two-day, 12–14-hour intensive track: learn the concept, predict a result, run a focused lab, validate evidence, then explain the production implications. The practical checkpoints are more important than command count.
