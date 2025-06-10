---
title: "When AWS DMS Drops Your Partitioned Table: Deep Dive into DETACH PARTITION with PostgreSQL"
date: 2025-06-03T20:00:00+05:30
draft: true
---

## 1. Introduction to AWS DMS & PostgreSQL

AWS Database Migration Service (DMS) is a managed service for migrating and replicating data with minimal downtime.  
With PostgreSQL as a source, DMS uses **logical replication slots** and the WAL (Write-Ahead Log) to perform both **full-load** and **CDC (ongoing)** migrations.  
This ensures transactional consistency, since changes are read from the WAL and replayed on the target.

## 2. Full-Load Parallelism vs. Single-Threaded CDC

- **Full-load** can be parallelized across tables or table slices using parameters like:
    - `MaxFullLoadSubTasks` (default 8, max 49) in **TargetMetadata**, and
    - `parallel-load` options in table-mapping rules.
      This greatly speeds up bulk data ingestion.

- **CDC**, however, preserves transaction order via a single stream through the **SORTER**.  
  It’s inherently single-threaded to maintain consistency—even if you’ve enabled multiple sub-tasks for full load.

## 3. Capturing DDL: `CaptureDdls` & Supported Statements

By default, DMS sets `CaptureDdls=true`, meaning **all** DDL events in the WAL are captured and replayed on the target.
Supported DDLs include: `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`, `RENAME`, etc.
Thus, operations you intend as “metadata-only” (e.g. DETACH PARTITION) can inadvertently translate into destructive DDL on your target.

## 4. How DMS Processes DDL & DML

1. **SOURCE_CAPTURE**  
   Reads WAL events (DML & DDL) via logical replication.
2. **SORTER**  
   Buffers and orders events in commit sequence. When in-memory buffers exceed `MemoryLimitTotal`, it spills to **swap files** on disk.
3. **TARGET_APPLY**  
   Applies the sorted stream to your target endpoint, including any captured DDL.

```plain
[SORTER] I: Swap files in use—total storage used has exceeded the in-memory limit…
```

## 5. The Partition DETACH Pitfall

Consider this common sequence:

```sql
ALTER TABLE parent_table DETACH PARTITION child_2025_06;
-- child_2025_06 now exists standalone
DROP TABLE child_2025_06;
```

On PostgreSQL alone, this detaches then drops. But with DMS’s DDL capture:

1. **DETACH** emits a DROP TABLE for the child partition.
2. **DMS** captures that DROP TABLE and replays it on the target—removing your data container unexpectedly. [Supported DDL](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.SupportedDDL.html)

## 6. Tuning Swap Files & Sorter Efficiency
When large or bursty change volumes hit the SORTER, it spills to disk. You can tune:

|  Setting | Purpose  |Default|
|---|---|---|
|  MemoryLimitTotal | Max MB for in-memory transaction cache before spill  |  1024 MB |
|  MemoryKeepTime | econds a transaction stays in memory before spilling  | 60 s  |
|  BatchApplyMemoryLimit | MB for batch-apply pre-processing           |  500 MB |
| BatchSplitSize  | Maximum change records per batch (0 = unlimited)  | 0  |

[Task Settings Params](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TaskSettings.ChangeProcessingTuning.html)

For extreme cases, see the AWS Knowledge Center on swap-file troubleshooting. [Swap file fixes](https://repost.aws/knowledge-center/dms-swap-files-consuming-space?utm_source=chatgpt.com)

## 7. Separate DDL from CDC

Use one task for full-load + CDC (no DDL) and another (or pg_dump) for schema changes only.
In your task JSON, set "CaptureDdls": false under PostgreSQLSettings when you only want DML.
docs.aws.amazon.com