# DP-800 Study Guide: CDC, CES, Change Tracking, Temporal Tables, and Ledger

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topic:** Detecting, retaining, streaming, and verifying SQL data changes  
> **Last reviewed:** October 5, 2026

## Exam objectives covered

This guide addresses the current DP-800 objectives to:

- Handle changes with Change Event Streaming (CES), Change Data Capture (CDC), Change Tracking, Azure Functions with the SQL trigger binding, and Azure Logic Apps.
- Choose an embedding-maintenance method, including table triggers, Change Tracking, Azure Functions, Logic Apps, CDC, and Microsoft Foundry.
- Design and implement specialized tables, including system-versioned temporal and ledger tables.
- Choose the correct feature for synchronization, event streaming, point-in-time history, or tamper evidence.

> [!IMPORTANT]
> CES is currently a preview feature and has platform-specific configuration and limitations. For an exam question, use the feature status, SQL platform, version, latency requirement, retained detail, and destination to choose the answer.

## 1. The core features in one table

| Feature | Main purpose | What it retains or emits | Consumption model |
|---|---|---|---|
| **CDC** | ETL, incremental loading, and change replay | Captured column values plus operation and LSN metadata | Consumer polls CDC table-valued functions |
| **CES** | Near-real-time event-driven integration | CloudEvents containing inserts, updates, deletes, schema, and old/new values | SQL pushes events to Azure Event Hubs or Fabric Eventstream |
| **Change Tracking** | Lightweight synchronization and cache/embedding refresh | Primary key, operation, version, and optional changed-column mask | Consumer polls `CHANGETABLE` and joins to the base table |
| **System-versioned temporal table** | Point-in-time history and row reconstruction | Complete previous row versions with validity periods | Query current and history tables with `FOR SYSTEM_TIME` |
| **Ledger table** | Cryptographically verifiable history and tamper evidence | Row history plus ledger transaction and hash-chain evidence | Query ledger views and verify against trusted digests |
| **DML trigger** | Immediate custom logic inside the modifying transaction | Whatever the trigger explicitly writes or enforces | Synchronous execution with the original DML statement |
| **Azure Functions SQL trigger** | Serverless reaction to SQL changes | Insert/update/delete batches derived through Change Tracking | Function runtime polls and invokes application code |
| **Azure Logic Apps SQL trigger** | Low-code workflow automation | Connector-specific inserted or changed row information | Managed or built-in connector starts a workflow |

### Fastest exam distinction

```text
Need old/new change values for incremental ETL?     → CDC
Need SQL to push events to Event Hubs/Eventstream?  → CES
Need only which rows changed since a checkpoint?    → Change Tracking
Need the row exactly as it existed at a past time?  → Temporal table
Need cryptographic evidence against tampering?      → Ledger table
Need custom logic in the same transaction?          → DML trigger
Need serverless code to react asynchronously?       → Azure Functions SQL trigger
Need a low-code business workflow?                  → Azure Logic Apps
```

## 2. Decision matrix

| Requirement | CDC | CES | Change Tracking | Temporal | Ledger |
|---|---:|---:|---:|---:|---:|
| Detect `INSERT`, `UPDATE`, and `DELETE` | Yes | Yes | Yes | Yes | Yes |
| Preserve previous row values | Yes, within retention | Included in supported event payloads | No | Yes | Yes for updatable ledger |
| Preserve intermediate updates | Yes with all-changes query | Events represent changes | No; changes are collapsed per row | Yes, except zero-duration versions need direct history access | Yes |
| Point-in-time query syntax | No | No | No | Yes | Updatable ledger is system-versioned |
| Push changes externally | No | Yes | No | No | No |
| Lightweight synchronization | Possible but more detail than needed | Possible but event architecture required | Best fit | Usually excessive | Not intended |
| Tamper-evident history | No | No | No | No | Yes |
| Requires consumer checkpoint | LSN | Event consumer offset/checkpoint | Change-tracking version | Time or business query boundary | Trusted digest/verification boundary |
| Normal source for embedding refresh | Yes | Yes | Yes | Usually not | No |

> [!TIP]
> Temporal tables answer **what did the complete row look like at a time?** Change Tracking answers **which keys changed after a version?** CDC answers **what row-value changes occurred in an LSN range?**

## 3. Change Data Capture

### Definition

Change Data Capture records inserts, updates, and deletes from the transaction log into relational change tables. It is designed for asynchronous consumers such as ETL pipelines, data warehouses, synchronization services, and embedding-maintenance jobs.

CDC exposes:

- captured source-column values;
- operation metadata;
- commit ordering through log sequence numbers (LSNs);
- before/after update images when requested; and
- net-change results when the capture instance is configured to support them.

### When to choose CDC

Choose CDC when the consumer needs:

- changed column values without adding application triggers;
- ordered, retained changes for an incremental pipeline;
- all intermediate changes or both update images;
- a replay window controlled by retention; or
- an LSN-based checkpoint.

Do not choose CDC merely to learn that a key changed. Change Tracking is lighter for that requirement.

### Enable CDC

Enable it first for the database, then for each source table:

```sql
USE ApplicationDb;
GO

EXEC sys.sp_cdc_enable_db;
GO

EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Product',
    @role_name = N'cdc_reader',
    @supports_net_changes = 1;
GO
```

Useful metadata:

```sql
SELECT name, is_cdc_enabled
FROM sys.databases
WHERE database_id = DB_ID();

SELECT name, is_tracked_by_cdc
FROM sys.tables
WHERE name = N'Product';

EXEC sys.sp_cdc_help_change_data_capture;
```

### Query an LSN range

```sql
DECLARE @from_lsn BINARY(10);
DECLARE @to_lsn   BINARY(10);

SET @from_lsn = sys.fn_cdc_get_min_lsn(N'dbo_Product');
SET @to_lsn   = sys.fn_cdc_get_max_lsn();

SELECT *
FROM cdc.fn_cdc_get_all_changes_dbo_Product
(
    @from_lsn,
    @to_lsn,
    N'all update old'
)
ORDER BY __$start_lsn, __$seqval;
```

The application should save the successfully processed high LSN and resume from a nonoverlapping next boundary.

### All changes versus net changes

| Function | Result |
|---|---|
| `cdc.fn_cdc_get_all_changes_<capture_instance>` | Every retained change in the LSN interval |
| `cdc.fn_cdc_get_net_changes_<capture_instance>` | One net result per changed row for the interval |

`fn_cdc_get_all_changes` supports:

- `all` — insert, delete, and the final image for an update;
- `all update old` — adds the old image for updates.

Net changes require a primary key or identified unique index and `@supports_net_changes = 1`.

### Operation codes

| `__$operation` | Meaning |
|---:|---|
| `1` | Delete |
| `2` | Insert |
| `3` | Update before image |
| `4` | Update after image |

`__$update_mask` indicates which captured columns changed.

### CDC strengths

- Captures values rather than only keys.
- Preserves commit order with LSN metadata.
- Supports incremental pull processing.
- Can retain intermediate changes.
- Does not add custom logic to each DML statement.

### CDC responsibilities

- Monitor capture and cleanup jobs or equivalent platform processes.
- Configure retention longer than the maximum consumer outage.
- Prevent the transaction log from growing because capture is delayed.
- Detect when a saved checkpoint is older than retained CDC data.
- Handle schema changes deliberately.
- Make consumers idempotent so replay does not duplicate downstream results.
- Keep checkpoint advancement in the same reliability boundary as downstream processing.

## 4. Change Event Streaming

### Definition

Change Event Streaming streams SQL data changes in near real time to Azure Event Hubs or a compatible Fabric Eventstream endpoint.

CES can publish:

- inserts, updates, and deletes;
- schema information;
- previous and new values where supported;
- transaction and ordering metadata; and
- CloudEvents serialized as JSON or Avro Binary.

### Architecture

```text
SQL transaction log
        ↓
CES stream group
        ↓
CloudEvents
        ↓
Azure Event Hubs / Fabric Eventstream
        ↓
Functions, stream processing, analytics, microservices, embeddings
```

### When to choose CES

Choose CES when:

- downstream systems should receive pushed events;
- latency should be near real time;
- multiple consumers need an event-streaming platform;
- the solution already uses Event Hubs or Fabric Real-Time Intelligence; or
- you want database-originated events without building a CDC poller.

### Core CES objects

| Object | Purpose |
|---|---|
| Stream group | Owns destination, authentication, serialization, and operational settings |
| Stream | Selects a source table and event options |
| Destination | Azure Event Hubs or a Fabric Eventstream-compatible endpoint |
| Consumer checkpoint | Records downstream progress in Event Hubs/Eventstream processing |

Conceptual configuration pattern:

```sql
EXEC sys.sp_create_change_stream_group
    @stream_group_name = N'product_changes',
    @destination_type = N'AzureEventHubs',
    @destination_location = N'<event-hubs-connection-information>',
    @serialization_format = N'JSON';
GO

EXEC sys.sp_create_change_stream
    @stream_group_name = N'product_changes',
    @source_schema = N'dbo',
    @source_name = N'Product';
GO
```

> [!NOTE]
> CES stored-procedure signatures and supported destination values can change during preview. Use the current documentation for implementation; focus on architecture and selection logic for the exam.

### Current platform facts

- CES applies to SQL Server 2025, Azure SQL Database, eligible Azure SQL Managed Instance configurations, and SQL database in Microsoft Fabric.
- It remains in preview.
- Azure Event Hubs and Fabric Eventstream are the intended destinations.
- New Azure SQL Database and SQL database in Fabric stream groups use the `AzureEventHubs` destination type, which uses the Kafka protocol.
- Managed identity support differs by platform and SQL Server servicing level.
- CES streams from the primary replica; failover planning is required.

### Delivery design

Design consumers for:

- duplicate delivery;
- retry and transient failure;
- consumer-group isolation;
- partition-aware ordering;
- schema evolution;
- poison events;
- replay from consumer checkpoints; and
- monitoring lag between commit and processed event.

Use a stable event identity or source transaction metadata to make downstream writes idempotent.

### CES versus CDC

| Question | CDC | CES |
|---|---|---|
| Who initiates delivery? | Consumer polls | SQL pushes to streaming destination |
| Destination | Relational CDC query functions | Event Hubs or Fabric Eventstream |
| Checkpoint | LSN | Consumer offset/checkpoint |
| Best fit | Batch/microbatch ETL and replay | Event-driven and real-time integration |
| Availability maturity | Established | Preview |

## 5. Change Tracking

### Definition

Change Tracking records that a row changed, not a complete history of row values. It is optimized for synchronization scenarios in which a consumer can retrieve the current row from the base table.

It records:

- the changed row's primary key;
- `I`, `U`, or `D` operation metadata;
- the change-tracking version;
- optional updated-column masks; and
- an optional change context.

### When to choose Change Tracking

Choose it for:

- mobile or offline synchronization;
- cache invalidation and refresh;
- search-index maintenance;
- embedding refresh when current row values are sufficient; and
- lightweight changed-key detection.

Do not choose it when every intermediate value or the old row image is required.

### Enable Change Tracking

```sql
ALTER DATABASE ApplicationDb
SET CHANGE_TRACKING = ON
(
    CHANGE_RETENTION = 7 DAYS,
    AUTO_CLEANUP = ON
);
GO

ALTER TABLE dbo.Product
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = ON);
GO
```

Tracked tables require a primary key.

### Retrieve changes

```sql
DECLARE @last_sync_version BIGINT = 100;
DECLARE @next_sync_version BIGINT;

SET @next_sync_version = CHANGE_TRACKING_CURRENT_VERSION();

SELECT
    CT.ProductId,
    CT.SYS_CHANGE_OPERATION,
    CT.SYS_CHANGE_VERSION,
    CT.SYS_CHANGE_COLUMNS,
    P.Name,
    P.Description,
    P.Price
FROM CHANGETABLE(CHANGES dbo.Product, @last_sync_version) AS CT
LEFT JOIN dbo.Product AS P
    ON P.ProductId = CT.ProductId;

-- Save @next_sync_version only after successful downstream processing.
```

Why `LEFT JOIN`? A deleted row no longer exists in the base table, but `CHANGETABLE` still reports its key and a `D` operation.

### Important functions

| Function | Purpose |
|---|---|
| `CHANGE_TRACKING_CURRENT_VERSION()` | Captures the latest committed version for the next checkpoint |
| `CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID(...))` | Tests whether a saved checkpoint is still valid |
| `CHANGETABLE(CHANGES table, version)` | Returns rows changed after a version |
| `CHANGETABLE(VERSION table, (key), (value))` | Returns the latest tracking metadata for one row |
| `CHANGE_TRACKING_IS_COLUMN_IN_MASK()` | Interprets the updated-column bitmask |
| `WITH CHANGE_TRACKING_CONTEXT` | Tags changes with an origin identifier |

### Validate the checkpoint

```sql
DECLARE @last_sync_version BIGINT = 100;
DECLARE @minimum_version BIGINT;

SET @minimum_version =
    CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID(N'dbo.Product'));

IF @last_sync_version < @minimum_version
BEGIN
    THROW 50001,
        'Checkpoint expired. Reinitialize the consumer.',
        1;
END;
```

If cleanup has removed needed metadata, incremental synchronization is no longer trustworthy. Reinitialize from the base table.

### Change Tracking behavior

- Several changes to one row are normally collapsed into the latest change metadata since the checkpoint.
- Current values come from the base table, not the tracking metadata.
- A delete supplies the key and operation but no old non-key values.
- `TRACK_COLUMNS_UPDATED` records which columns were targeted by updates, not a complete before/after value history.
- Retention must exceed the maximum consumer outage.

## 6. System-versioned temporal tables

### Definition

A system-versioned temporal table maintains complete row history automatically. SQL Server manages a validity period for each version and moves previous versions into a history table after updates and deletes.

```text
Current temporal table
├─ current row values
├─ ValidFrom
└─ ValidTo = 9999-12-31...

History table
├─ previous row values
├─ period start
└─ period end
```

Temporal tables are ideal for:

- point-in-time reconstruction;
- tracking how values evolved;
- recovering from accidental changes;
- trend analysis;
- slowly changing dimensions; and
- data forensics.

### Create a temporal table

```sql
CREATE SCHEMA History;
GO

CREATE TABLE dbo.ProductPrice
(
    ProductId   INT NOT NULL PRIMARY KEY,
    ProductName NVARCHAR(200) NOT NULL,
    Price       DECIMAL(12,2) NOT NULL,

    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH
(
    SYSTEM_VERSIONING = ON
    (
        HISTORY_TABLE = History.ProductPriceHistory,
        DATA_CONSISTENCY_CHECK = ON
    )
);
GO
```

The period columns:

- must use `DATETIME2`;
- are maintained by the Database Engine;
- use UTC transaction-begin time; and
- can be marked `HIDDEN` so older applications using `SELECT *` or positional inserts are less disrupted.

### What happens to each DML operation

| Operation | Current table | History table |
|---|---|---|
| `INSERT` | New current version is inserted | Nothing |
| `UPDATE` | New values become current | Previous version is copied to history |
| `DELETE` | Current row is removed | Deleted version is copied to history |

All rows changed in one transaction use the transaction begin time for the temporal period.

### Query temporal data

Current data only:

```sql
SELECT *
FROM dbo.ProductPrice
WHERE ProductId = 42;
```

State at one point in time:

```sql
SELECT *
FROM dbo.ProductPrice
FOR SYSTEM_TIME AS OF '2026-09-01T12:00:00'
WHERE ProductId = 42;
```

All versions:

```sql
SELECT *
FROM dbo.ProductPrice
FOR SYSTEM_TIME ALL
WHERE ProductId = 42
ORDER BY ValidFrom;
```

Versions active during a range:

```sql
SELECT *
FROM dbo.ProductPrice
FOR SYSTEM_TIME
    FROM '2026-09-01T00:00:00'
    TO   '2026-10-01T00:00:00'
WHERE ProductId = 42;
```

### `FOR SYSTEM_TIME` choices

| Clause | Meaning |
|---|---|
| `AS OF t` | Version valid at time `t` |
| `FROM a TO b` | Versions active in half-open interval; upper boundary excluded |
| `BETWEEN a AND b` | Like `FROM ... TO`, but includes versions beginning at the upper boundary |
| `CONTAINED IN (a, b)` | Versions whose complete validity period falls inside the range |
| `ALL` | Current and historical versions |

### Temporal retention and performance

- History grows with update/delete frequency, row width, and retention duration.
- Create indexes that match point-in-time and entity-history queries.
- Consider history retention policies and partitioning for large histories.
- Avoid querying the history table directly unless a specialized case requires it; `FOR SYSTEM_TIME` combines current and history correctly.
- Multiple updates to the same key in one transaction can create zero-duration versions that ordinary temporal queries filter out.
- Updating a row even without changing column values can create a history version.

### Temporal limitations and responsibilities

- Temporal history is not cryptographically protected.
- A sufficiently privileged user can disable system versioning and alter history.
- Temporal tables do not inherently record the application user, business reason, or correlation ID; add explicit columns if needed.
- Turning `SYSTEM_VERSIONING = OFF` stops automatic history capture but does not automatically drop the history table.
- Schema changes and history cleanup require planning.
- Security policies on a current table do not automatically secure its history table; apply appropriate permissions or RLS to both.

### Temporal versus CDC

| Question | Temporal | CDC |
|---|---|---|
| Main query key | Wall-clock point or time range | LSN range |
| Stores full prior row versions | Yes | Captured change values |
| Intended for point-in-time state | Yes | No |
| Intended for ETL/change delivery | Not primarily | Yes |
| History table maintained automatically | Yes | CDC change tables maintained by capture/cleanup |
| Tamper-evident | No | No |

### Temporal versus Change Tracking

| Question | Temporal | Change Tracking |
|---|---|---|
| Complete historical values | Yes | No |
| Lightweight changed-key list | No | Yes |
| Deleted non-key values available | Yes | No |
| Synchronization checkpoint | Time/query boundary | Database version |

## 7. Ledger tables

### Definition

SQL ledger provides tamper evidence by maintaining cryptographically linked data history. Database-ledger digests can be stored outside the database and later used to verify that ledger data has not been altered.

Ledger does not replace permissions, encryption, backups, or change-delivery features.

### Ledger table types

| Type | Allowed DML | History behavior | Best fit |
|---|---|---|---|
| **Updatable ledger table** | `INSERT`, `UPDATE`, `DELETE` | Automatically system-versioned; old versions enter a history table | Records that change but need verifiable history |
| **Append-only ledger table** | `INSERT` only | No separate version history is needed | Facts that must never be updated or deleted through normal database operations |

### Updatable ledger table

```sql
CREATE SCHEMA Compliance;
GO

CREATE TABLE Compliance.AccountBalance
(
    AccountId INT NOT NULL PRIMARY KEY,
    Balance   DECIMAL(18,2) NOT NULL
)
WITH
(
    SYSTEM_VERSIONING = ON,
    LEDGER = ON
);
GO
```

An updatable ledger table combines temporal versioning with ledger metadata and cryptographic integrity evidence.

### Append-only ledger table

```sql
CREATE TABLE Compliance.FinancialEvent
(
    EventId    BIGINT IDENTITY PRIMARY KEY,
    EventType  NVARCHAR(100) NOT NULL,
    Amount     DECIMAL(18,2) NOT NULL,
    OccurredAt DATETIME2 NOT NULL
)
WITH
(
    LEDGER = ON (APPEND_ONLY = ON)
);
GO
```

Attempts to update or delete rows through the database API are rejected.

### Ledger view and transaction metadata

The generated ledger view combines current and historical ledger rows and exposes operation metadata. It can be joined to `sys.database_ledger_transactions` for transaction information.

```sql
SELECT
    L.*,
    T.commit_time,
    T.principal_name,
    L.ledger_operation_type_desc
FROM Compliance.AccountBalance_Ledger AS L
JOIN sys.database_ledger_transactions AS T
    ON T.transaction_id = L.ledger_transaction_id;
```

Use the generated ledger view instead of querying the ledger history table directly.

### Digests and verification

```text
Ledger rows
   ↓ hashed
Transaction Merkle roots
   ↓ collected into blocks
Hash-linked database ledger
   ↓ digest exported outside database
Trusted storage such as Azure Storage or Azure Confidential Ledger
```

Verification recomputes ledger hashes and compares them with trusted external digests:

```sql
SELECT *
FROM sys.database_ledger_digest_locations;
GO

EXEC sys.sp_verify_database_ledger_from_digest_storage;
GO
```

A mismatch is evidence that the protected data or ledger history may have been altered.

> [!IMPORTANT]
> Ledger is **tamper-evident**, not magically tamper-proof. The external digest must be protected independently of the database.

### Temporal versus ledger

| Requirement | Temporal | Updatable ledger |
|---|---|---|
| Point-in-time row history | Yes | Yes |
| Normal update/delete support | Yes | Yes |
| Cryptographic tamper evidence | No | Yes |
| External trusted digest | No | Recommended/required for independent verification |
| Simpler history and lower integrity overhead | Better fit | More overhead |

Choose temporal for ordinary point-in-time history. Choose ledger when verifiable integrity is a requirement.

## 8. DML triggers

A DML trigger runs automatically for `INSERT`, `UPDATE`, or `DELETE`. The trigger and the triggering statement run in the same transaction.

### When triggers fit

- Immediate enforcement that constraints cannot express.
- Writing a small transactional outbox or work queue.
- Maintaining derived data synchronously.
- Capturing application-specific context explicitly.

### When triggers are risky

- They increase DML latency and locking time.
- An error can roll back the original transaction.
- Hidden logic can surprise application developers.
- Recursive or nested behavior can become complex.
- External network calls inside a trigger make the transaction fragile.
- Triggers fire once per statement, not once per row; code must handle multirow `inserted` and `deleted` sets.

### Transactional work-queue pattern

```sql
CREATE TABLE dbo.EmbeddingWorkQueue
(
    WorkId      BIGINT IDENTITY PRIMARY KEY,
    DocumentId  BIGINT NOT NULL,
    Operation   VARCHAR(10) NOT NULL,
    EnqueuedAt  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ProcessedAt DATETIME2 NULL
);
GO

CREATE TRIGGER dbo.tr_Document_EmbeddingQueue
ON dbo.Document
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Inserts and updates produce an UPSERT task.
    INSERT dbo.EmbeddingWorkQueue(DocumentId, Operation)
    SELECT i.DocumentId, 'UPSERT'
    FROM inserted AS i;

    -- Rows found only in deleted are actual deletes.
    INSERT dbo.EmbeddingWorkQueue(DocumentId, Operation)
    SELECT d.DocumentId, 'DELETE'
    FROM deleted AS d
    LEFT JOIN inserted AS i
        ON i.DocumentId = d.DocumentId
    WHERE i.DocumentId IS NULL;
END;
GO
```

The trigger performs only a fast local insert. A separate worker calls the embedding model and updates the vector. This is safer than calling an external model endpoint from inside the user transaction.

## 9. Azure Functions with the SQL trigger binding

The Azure SQL trigger for Functions reacts to inserted, updated, and deleted rows. It uses SQL Change Tracking internally.

### Required database setup

```sql
ALTER DATABASE ApplicationDb
SET CHANGE_TRACKING = ON
(
    CHANGE_RETENTION = 2 DAYS,
    AUTO_CLEANUP = ON
);
GO

ALTER TABLE dbo.Document
ENABLE CHANGE_TRACKING;
GO
```

The function identity needs:

- `SELECT` on the monitored table;
- `VIEW CHANGE TRACKING` on the table; and
- permissions for the binding's internal `az_func` schema and state/lease tables.

Use managed identity when possible.

### Conceptual C# function

```csharp
[Function("DocumentChanged")]
public async Task Run(
    [SqlTrigger("dbo.Document", "SqlConnectionString")]
    IReadOnlyList<SqlChange<Document>> changes)
{
    foreach (SqlChange<Document> change in changes)
    {
        // Make processing idempotent.
        // Generate, update, or delete the embedding as required.
    }
}
```

Each change includes the row-shaped item and an operation such as insert, update, or delete.

### Design points

- Retention must cover function downtime.
- The binding keeps internal progress/lease state.
- Batch size and polling interval are configurable.
- Make handlers idempotent and safe for retry.
- Do not assume encrypted values are automatically decrypted in the payload.
- Grant the function identity only the permissions it needs.

## 10. Azure Logic Apps

Azure Logic Apps can use SQL connector triggers and actions to create low-code workflows.

Use it when a change should start a workflow such as:

- send an approval request;
- notify Teams or email;
- create a ticket;
- call another SaaS connector;
- run a stored procedure; or
- coordinate a small business process.

Important distinctions:

- Many SQL connector triggers poll on a configured schedule.
- Trigger availability and semantics differ between Consumption and Standard workflows and between built-in and managed connectors.
- A trigger starts the workflow; actions perform the downstream work.
- Use managed identity or protected connections when available.
- Configure concurrency, retries, and duplicate-safe actions.
- Logic Apps is orchestration, not a relational row-history store.

## 11. Choosing an embedding-maintenance method

| Requirement | Recommended starting point | Why |
|---|---|---|
| Immediate transactional enqueue | DML trigger + work queue | Queue record commits atomically with the source change |
| Serverless custom code | Azure Functions SQL trigger | Change Tracking-backed trigger invokes code and can scale |
| Near-real-time streaming to many consumers | CES | Pushes events through Event Hubs/Eventstream |
| Incremental batch with old/new values | CDC | Values and LSN replay are suited to ETL-style processing |
| Lightweight periodic refresh | Change Tracking | Efficient changed-key enumeration |
| Low-code workflow | Azure Logic Apps | Connector-based orchestration |
| Rebuild data as it looked at a past time | Temporal table | `FOR SYSTEM_TIME AS OF` reconstructs state |
| Cryptographic evidence for protected records | Ledger | Verifiable history, but not a change-delivery system |

### Reliable embedding workflow

```text
Source change
    ↓
CDC / CT / CES / queue checkpoint
    ↓
Load current source text
    ↓
Chunk and generate embedding
    ↓
Upsert vector with source-version metadata
    ↓
Commit downstream result
    ↓
Advance checkpoint
```

Reliability rules:

- Derive an idempotency key from the entity key plus source version/LSN/event identity.
- Do not advance the checkpoint until downstream storage succeeds.
- Delete or invalidate embeddings when the source row is deleted.
- Re-embed when model, dimensions, chunking, or relevant text changes.
- Store model name/version and source-change version beside the vector.
- Monitor lag and dead-letter or poison work.
- Support a full rebuild when retention is exceeded or processing logic changes.

## 12. One update viewed by every feature

Suppose product `42` changes from price `10.00` to `12.00`.

### CDC sees

- an update operation;
- commit LSN and sequence value;
- new values; and
- optionally the before image and update mask.

### CES sees

- a near-real-time CloudEvent;
- operation and source metadata;
- schema and old/new values according to configuration; and
- a destination consumer offset for processing.

### Change Tracking sees

- primary key `42`;
- operation `U`;
- a database version; and
- optionally a changed-column mask.

It does not retain `10.00`; the consumer reads the current `12.00` from the base table.

### Temporal sees

- the previous full row with `10.00` in the history table and a closed validity period; and
- the current full row with `12.00` and an open validity period.

### Updatable ledger sees

- temporal-style current and previous versions;
- ledger operation and transaction metadata; and
- cryptographic linkage that can be verified against a trusted digest.

### DML trigger sees

- `deleted.Price = 10.00` and `inserted.Price = 12.00` during the same transaction; and
- performs the custom set-based action coded by the developer.

## 13. Related SQL features: should you add them?

### Recommended additions for DP-800

These are included in this guide because the blueprint names them directly:

1. **DML triggers** — especially the transactional queue/outbox pattern for embedding maintenance.
2. **Azure Functions SQL trigger binding** — serverless change handling backed by Change Tracking.
3. **Azure Logic Apps** — low-code change-driven workflows.

### Useful comparisons, but lower exam priority

| Feature | What it solves | Why it is not a direct replacement |
|---|---|---|
| Transactional replication | Low-latency server-to-server distribution with transaction order | Designed to maintain subscribers, not provide general application change history |
| Query Notifications / `SqlDependency` | Tells an application that a qualifying query result may have changed | Invalidates a query result; it does not provide row changes or history |
| Service Broker | Durable in-database messaging and queues | Powerful but requires custom message, queue, activation, and security design |
| `OUTPUT` clause | Returns affected rows from one DML statement | Only the executing statement receives the rows; it is not a persistent change feed |
| Event notifications | Reports selected DDL and trace events through Service Broker | Not a row-level DML change feed |

For this DP-800 topic, learn the three recommended additions deeply and recognize the lower-priority features by purpose.

## 14. Common exam traps

1. **CDC and Change Tracking are not synonyms.** CDC retains values; Change Tracking mainly retains keys and metadata.
2. **Change Tracking does not preserve old values.** Read current values from the base table.
3. **Deleted Change Tracking rows require a `LEFT JOIN`.** The base row no longer exists.
4. **Validate `CHANGE_TRACKING_MIN_VALID_VERSION`.** Reinitialize if the checkpoint expired.
5. **Do not advance a checkpoint before downstream success.** That can lose work.
6. **CDC all changes and net changes are different.** Choose based on whether intermediate changes matter.
7. **CDC checkpoints are LSNs; Change Tracking checkpoints are versions.** They are not interchangeable.
8. **CES is push-based; CDC and Change Tracking are normally pull-based.**
9. **CES consumer checkpoints live in the streaming architecture.** They are not CDC LSNs.
10. **Temporal is for point-in-time row history, not event delivery.**
11. **Temporal history is not tamper-evident.** Use ledger when cryptographic verification is required.
12. **Temporal times use UTC transaction-begin time.** They are not arbitrary business-valid dates.
13. **Updatable ledger is also system-versioned.** Append-only ledger is insert-only.
14. **Ledger is not a normal embedding-change feed.** Use CT, CDC, CES, Functions, Logic Apps, or a queue.
15. **Triggers run in the original transaction.** Slow or failing trigger logic affects the user's DML.
16. **Triggers must handle multiple rows.** Never assume one row per trigger invocation.
17. **Azure Functions SQL trigger uses Change Tracking.** Its retention and permissions still matter.
18. **Logic Apps is workflow orchestration, not a history table.**
19. **Transactional replication distributes data; it is not temporal history.**
20. **Query Notifications indicate invalidation, not the exact changed rows.**

## 15. Memory aids

### Feature mnemonic

```text
CDC       = Change Data Copies
CES       = Change Events Stream
CT        = Changed keys Tracked
Temporal  = Time-travel rows
Ledger    = Linked hashes
Trigger   = Transaction-time logic
```

### Checkpoints

```text
CDC              → LSN
CES              → event consumer offset/checkpoint
Change Tracking  → database version
Temporal         → UTC time or time range
Ledger           → trusted digest and verification range
Functions trigger→ binding lease/state backed by Change Tracking
```

## 16. Practice questions

### Question 1

An ETL pipeline must retrieve old and new values for every update in a retained interval. Which feature is the best fit?

A. Change Tracking  
B. CDC  
C. Query Notifications  
D. Temporal `AS OF` only

<details>
<summary>Answer</summary>

**B.** CDC can return every change and both update images with `all update old`.

</details>

### Question 2

A service needs only the keys of rows changed since its last synchronization and can read current values from the table. Which feature should it use?

A. Change Tracking  
B. Temporal  
C. Append-only ledger  
D. Transactional replication

<details>
<summary>Answer</summary>

**A.** Change Tracking is optimized for lightweight changed-key synchronization.

</details>

### Question 3

What should a Change Tracking consumer do when its saved version is lower than `CHANGE_TRACKING_MIN_VALID_VERSION`?

A. Continue normally  
B. Convert the version to an LSN  
C. Reinitialize from a fresh baseline  
D. Query the temporal history table

<details>
<summary>Answer</summary>

**C.** Required tracking metadata has been cleaned up, so an incremental result is no longer reliable.

</details>

### Question 4

Changes must be pushed in near real time to Azure Event Hubs for multiple consumers. Which feature is the direct fit?

A. CES  
B. Temporal  
C. Change Tracking alone  
D. Query Notifications

<details>
<summary>Answer</summary>

**A.** CES streams database change events to Event Hubs or Fabric Eventstream.

</details>

### Question 5

Which statement about CES is currently correct?

A. It is a mature replacement for every CDC workload  
B. It is a preview feature with platform-specific limitations  
C. It writes directly to any Kafka cluster  
D. It uses `FOR SYSTEM_TIME`

<details>
<summary>Answer</summary>

**B.** CES remains in preview, so platform and destination details matter.

</details>

### Question 6

An analyst must reconstruct a product row exactly as it appeared at noon last Tuesday. Which feature should be used?

A. Change Tracking  
B. `FOR SYSTEM_TIME AS OF` on a temporal table  
C. Query Notifications  
D. A CES consumer offset

<details>
<summary>Answer</summary>

**B.** Temporal tables preserve row versions with validity periods for point-in-time queries.

</details>

### Question 7

What happens to the old version when a system-versioned temporal row is updated?

A. It is discarded  
B. It is sent to Event Hubs  
C. It is copied to the history table with a closed validity period  
D. It becomes a Change Tracking checkpoint

<details>
<summary>Answer</summary>

**C.** The previous full row version moves into the history table.

</details>

### Question 8

Which temporal clause returns current and all historical versions?

A. `FOR SYSTEM_TIME ALL`  
B. `CHANGETABLE(ALL)`  
C. `FOR LEDGER ALL`  
D. `CDC ALL`

<details>
<summary>Answer</summary>

**A.** `FOR SYSTEM_TIME ALL` unions qualifying current and history rows.

</details>

### Question 9

A regulator requires cryptographic evidence that protected row history has not been altered. Which feature is the best fit?

A. Temporal table alone  
B. Change Tracking  
C. Ledger table with protected external digests  
D. DML trigger alone

<details>
<summary>Answer</summary>

**C.** Ledger provides hash-linked history that can be verified against trusted digests.

</details>

### Question 10

Records may be inserted but must not be updated or deleted through the database API. Which table type is designed for this?

A. Updatable ledger  
B. Append-only ledger  
C. Temporal table  
D. CDC change table

<details>
<summary>Answer</summary>

**B.** Append-only ledger tables accept inserts and reject updates and deletes.

</details>

### Question 11

Which feature combines normal updates and deletes, temporal-style history, and cryptographic verification?

A. Updatable ledger table  
B. Change Tracking  
C. Query Notifications  
D. Logic Apps

<details>
<summary>Answer</summary>

**A.** An updatable ledger table is system-versioned and adds ledger integrity evidence.

</details>

### Question 12

Why should an embedding trigger normally enqueue work rather than call an external model directly?

A. Triggers cannot read `inserted`  
B. External calls would lengthen and destabilize the original transaction  
C. Triggers run only once per database  
D. Queues disable transactions

<details>
<summary>Answer</summary>

**B.** The trigger runs in the user's transaction, so slow or failing network work can delay or roll back the DML.

</details>

### Question 13

What is a critical rule for DML trigger implementation?

A. Assume one row per invocation  
B. Handle `inserted` and `deleted` as multirow sets  
C. Store the checkpoint as an Event Hubs offset  
D. Disable constraints first

<details>
<summary>Answer</summary>

**B.** One statement can affect many rows, and the trigger fires per statement.

</details>

### Question 14

What does the Azure Functions SQL trigger use to detect changes?

A. Temporal history  
B. SQL Change Tracking  
C. Transactional replication  
D. Ledger digests

<details>
<summary>Answer</summary>

**B.** Change Tracking must be enabled on the database and monitored table.

</details>

### Question 15

A business analyst wants a low-code workflow that sends an approval when a SQL row is created. Which option is the best starting point?

A. Azure Logic Apps SQL trigger  
B. Ledger verification  
C. CDC cleanup job  
D. Query Store

<details>
<summary>Answer</summary>

**A.** Logic Apps provides connector-based low-code orchestration.

</details>

### Question 16

Which feature is intended to keep server-to-server subscriber data synchronized in near real time while preserving transaction order?

A. Transactional replication  
B. Query Notifications  
C. Temporal tables  
D. DDM

<details>
<summary>Answer</summary>

**A.** Transactional replication distributes changes to subscribers in transaction order.

</details>

### Question 17

An application only needs to know that the result of a qualifying query may have changed so it can invalidate a cache. Which lower-priority feature fits?

A. Query Notifications / `SqlDependency`  
B. CDC all changes  
C. Ledger verification  
D. Temporal `CONTAINED IN`

<details>
<summary>Answer</summary>

**A.** Query Notifications signal possible result invalidation; they do not provide exact changed rows.

</details>

### Question 18

When should a consumer advance its checkpoint?

A. Before reading changes  
B. Immediately after receiving a batch, before processing  
C. Only after the downstream operation succeeds durably  
D. Only when temporal system versioning is disabled

<details>
<summary>Answer</summary>

**C.** Advancing earlier can lose work after a failure. Idempotency allows safe replay if failure occurs before checkpoint advancement.

</details>

## 17. Rapid-review checklist

- [ ] I can distinguish CDC values and LSNs from Change Tracking keys and versions.
- [ ] I know all changes versus net changes in CDC.
- [ ] I can validate Change Tracking retention with `CHANGE_TRACKING_MIN_VALID_VERSION`.
- [ ] I know CES pushes CloudEvents to Event Hubs or Fabric Eventstream.
- [ ] I know CES is currently preview and platform-specific.
- [ ] I can create and query a system-versioned temporal table.
- [ ] I know all five `FOR SYSTEM_TIME` forms.
- [ ] I understand that temporal history is not tamper-evident.
- [ ] I can distinguish updatable and append-only ledger tables.
- [ ] I know external ledger digests enable independent verification.
- [ ] I understand trigger transaction and multirow behavior.
- [ ] I know the Azure Functions SQL trigger depends on Change Tracking.
- [ ] I can choose Logic Apps for low-code workflows.
- [ ] I can select an embedding-maintenance method from latency, detail, and reliability requirements.
- [ ] I recognize transactional replication and Query Notifications without confusing them with row history.

## 18. Official Microsoft references

- [DP-800 official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [Change Data Capture overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver17)
- [Work with CDC change data](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/work-with-change-data-sql-server?view=sql-server-ver17)
- [Change Event Streaming overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview?view=sql-server-ver17)
- [CES frequently asked questions](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/frequently-asked-questions-faq?view=sql-server-ver17)
- [Change Tracking overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-tracking-sql-server?view=sql-server-ver17)
- [Work with Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/work-with-change-tracking-sql-server?view=sql-server-ver17)
- [Temporal tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal/overview?view=sql-server-ver17)
- [Query temporal data](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal/query-data?view=sql-server-ver17)
- [Temporal usage scenarios](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal/usage-scenarios?view=sql-server-ver17)
- [Ledger overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview?view=sql-server-ver17)
- [DML triggers](https://learn.microsoft.com/en-us/sql/relational-databases/triggers/dml-triggers?view=sql-server-ver17)
- [Azure Functions SQL trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-azure-sql-trigger)
- [Azure Logic Apps SQL connector](https://learn.microsoft.com/en-us/azure/connectors/connectors-create-api-sqlazure)
- [Transactional replication](https://learn.microsoft.com/en-us/sql/relational-databases/replication/transactional/transactional-replication?view=sql-server-ver17)
- [Query Notifications](https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/enable-query-notifications?view=sql-server-ver17)

---

## Final exam summary

> **CDC retains change values; CES streams change events; Change Tracking identifies changed keys; temporal tables preserve point-in-time row history; ledger adds cryptographic integrity; triggers, Functions, and Logic Apps execute reactions.**
