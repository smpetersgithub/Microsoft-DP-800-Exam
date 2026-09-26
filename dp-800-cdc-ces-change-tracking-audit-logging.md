# DP-800 Study Guide: CDC, CES, Change Tracking, Ledger Tables, and Audit Logging

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topic:** Detecting, retaining, streaming, and auditing SQL changes  
> **Last reviewed:** September 26, 2026

## Exam objectives covered

This guide addresses DP-800 scenarios that require you to:

- Handle database changes with Change Data Capture (CDC), Change Event Streaming (CES), and Change Tracking.
- Select an embedding-maintenance method when source data changes.
- Integrate SQL solutions with event-driven Azure and Microsoft Fabric services.
- Design audit logging for security, monitoring, investigations, and compliance.
- Choose between updatable and append-only ledger tables for tamper-evident data.
- Distinguish data synchronization from security auditing.

> [!IMPORTANT]
> Feature availability and implementation details differ among SQL Server, Azure SQL Database, Azure SQL Managed Instance, and SQL database in Microsoft Fabric. CES is currently a preview feature. In an exam question, use the named platform and requirements to eliminate unsupported choices.

## 1. The five features in one table

| Feature | Primary purpose | What it captures | Where information goes | Typical consumer |
|---|---|---|---|---|
| **CDC** | Incremental data movement and processing | Row values for inserts, updates, and deletes; can expose before/after update values | Change tables in the source database | ETL process, polling application, embedding batch job |
| **CES** | Near-real-time, event-driven integration | Insert, update, and delete events with current schema and previous/new values | Azure Event Hubs or Fabric Eventstream as CloudEvents | Functions, stream processors, microservices, embedding event pipeline |
| **Change Tracking** | Lightweight synchronization | Changed primary keys, operation, version, and optional changed-column mask | Internal change-tracking tables | Sync client, cache refresher, periodic embedding job |
| **Audit logging** | Security and compliance evidence | Selected user, login, server, database, and object actions | Audit files/logs, Azure Storage, Log Analytics, or Event Hubs | Security operations, compliance, investigation |
| **Ledger table** | Cryptographic tamper evidence and protected history | Ledger-table transactions, row versions, transaction identity, and hash-chain evidence | Ledger/history tables, ledger views, and externally protected database digests | Auditor, forensic investigator, high-integrity system of record |

### Fastest exam distinction

```text
Need every row change and values?      → CDC
Need external near-real-time events?   → CES
Need only which rows changed?          → Change Tracking
Need who performed an action?          → Audit logging
Need to prove data was not tampered?   → Ledger table
```

## 2. Decision matrix

| Requirement | CDC | CES | Change Tracking | Audit logging | Ledger table |
|---|:---:|:---:|:---:|:---:|:---:|
| Detect `INSERT`, `UPDATE`, and `DELETE` | Yes | Yes | Yes | Can log selected statements/actions | Yes for ledger data |
| Preserve row values | Yes | Yes, in emitted events | No; query current base-table values | Not intended as a row-version store | Updatable ledger preserves previous versions |
| Preserve intermediate changes | Yes, within retention | Events are emitted per change | No; changes are consolidated relative to a baseline | Logs selected actions, not a replication feed | Yes for updatable ledger history |
| Before/after update values | Yes, when querying all changes with old values | Yes | No | Not the correct feature | Current row plus protected historical versions |
| Push changes externally | No; consumer polls | Yes | No; consumer polls | Azure audit can write to external targets | Digests should be stored externally; row events aren't pushed |
| Lightweight synchronization | More overhead | External streaming infrastructure required | Best fit | No | No |
| ETL/incremental warehouse load | Strong fit | Possible through streaming | Only if keys/current state are enough | No | Not the primary purpose |
| Event-driven microservices | Requires a polling/publishing layer | Strong fit | Requires a polling/publishing layer | No | No |
| Keep embeddings current | Strong for batch/history needs | Strong for near-real-time events | Strong for lightweight periodic refresh | No | Not the primary purpose |
| Security/compliance evidence | Not its primary purpose | Not a substitute for SQL Audit | No | Records selected actions and actors | Proves data integrity and detects tampering |

## 3. Change Data Capture (CDC)

### Definition

CDC reads committed changes from the transaction log and writes change information into relational **change tables** in the `cdc` schema. Consumers query table-valued functions over a log sequence number (LSN) range.

```text
INSERT / UPDATE / DELETE
          ↓
    Transaction log
          ↓
     CDC capture process
          ↓
   cdc.<capture_instance>_CT
          ↓
 Consumer queries an LSN range
```

### When to choose CDC

Use CDC when the application needs:

- incremental ETL or ELT;
- a relational representation of source changes;
- delete values or update before/after images;
- every retained change rather than only the latest changed key;
- a polling-based embedding maintenance process that may need source values or history.

### How CDC is organized

1. Enable CDC at the database level.
2. Enable CDC on individual source tables.
3. Enabling a table creates a **capture instance**.
4. The capture instance includes a change table and query functions.
5. A cleanup process removes data beyond the configured retention window.

The default capture-instance name is normally `<schema>_<table>`. A source table can have no more than two active capture instances.

### Enable CDC

```sql
-- Run in the target database.
EXEC sys.sp_cdc_enable_db;
GO

EXEC sys.sp_cdc_enable_table
    @source_schema        = N'dbo',
    @source_name          = N'Documents',
    @role_name            = N'cdc_readers',
    @supports_net_changes = 1;
GO
```

`@role_name` creates or uses a gating role that limits access to captured data. Use `@captured_column_list` if only selected columns should be captured.

### Query an LSN range

```sql
DECLARE @FromLsn BINARY(10) =
    sys.fn_cdc_get_min_lsn(N'dbo_Documents');

DECLARE @ToLsn BINARY(10) =
    sys.fn_cdc_get_max_lsn();

SELECT *
FROM cdc.fn_cdc_get_all_changes_dbo_Documents
(
    @FromLsn,
    @ToLsn,
    N'all update old'
);
```

The generated function name depends on the capture-instance name.

### All changes versus net changes

| Query mode | Meaning |
|---|---|
| `fn_cdc_get_all_changes_<capture_instance>` | Returns each retained change in the requested LSN range |
| `fn_cdc_get_net_changes_<capture_instance>` | Returns one net result per source row across the LSN range |

Net-change queries require `@supports_net_changes = 1` when the capture instance is created and require a primary key or suitable unique index.

### CDC operation codes

| `__$operation` | Meaning |
|---:|---|
| `1` | Delete row |
| `2` | Insert row |
| `3` | Update before-image |
| `4` | Update after-image |

Other important metadata includes:

- `__$start_lsn`: orders committed transactions.
- `__$seqval`: orders multiple changes within a transaction.
- `__$update_mask`: identifies captured columns affected by an update.

### Platform behavior

- **SQL Server and Azure SQL Managed Instance:** SQL Server Agent capture and cleanup jobs process CDC.
- **Azure SQL Database:** an internal database-scoped scheduler replaces SQL Server Agent jobs.
- Azure SQL Database supports CDC on all vCore service tiers; DTU-based databases require S3 or higher.

### CDC advantages

- Provides relational, queryable change data.
- Supports before/after update values.
- Good fit for incremental ETL and replay within the retention window.
- Consumers can process ranges independently using LSN checkpoints.

### CDC limitations and responsibilities

- Change tables consume database storage.
- Capture lag or failures can delay transaction-log truncation and increase log size.
- Consumers must save their processed LSN checkpoint.
- Cleanup/retention must exceed the longest expected consumer outage.
- Schema changes require planning because capture instances reflect captured-column metadata.
- CDC is not a security audit and does not answer every “who executed this?” question.
- CES cannot be enabled on a database that is already enabled for CDC.

## 4. Change Event Streaming (CES)

### Definition

CES streams SQL `INSERT`, `UPDATE`, and `DELETE` changes directly to **Azure Event Hubs** or **Fabric Eventstream** in near real time.

Events follow the **CloudEvents** specification and can be serialized as native JSON or Avro binary.

```text
SQL DML change
      ↓
Transaction log
      ↓
CES stream group
      ↓
Azure Event Hubs or Fabric Eventstream
      ↓
One or more independent consumers
```

### When to choose CES

Use CES for:

- event-driven architectures;
- near-real-time embedding generation;
- microservice or distributed-system synchronization;
- real-time analytics;
- multiple downstream consumers;
- loose coupling between the database and processing applications.

### Core objects

| Object | Purpose |
|---|---|
| Stream group | Defines destination, credential, message size, partitioning, and tracked tables |
| Object | A tracked SQL table in a stream group |
| Destination | Azure Event Hubs or Fabric Eventstream |
| CloudEvent | Standard event envelope carrying the change data |

### CES event information

CES events can contain:

- operation type: `INS`, `UPD`, or `DEL`;
- current table schema;
- previous and new values;
- identifiers used to order segments of a large logical message;
- CloudEvents metadata such as event type and ID.

### Conceptual Azure SQL configuration

```sql
-- Values in angle brackets are deployment-specific placeholders.
USE <DatabaseName>;
GO

CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<StrongPassword>';
GO

CREATE DATABASE SCOPED CREDENTIAL <CesCredential>
    WITH IDENTITY = 'Managed Identity';
GO

EXEC sys.sp_enable_event_stream;
GO

EXEC sys.sp_create_event_stream_group
    @stream_group_name      = N'DocumentChanges',
    @destination_type       = N'AzureEventHubs',
    @destination_location   = N'<namespace>.servicebus.windows.net:9093/<event-hub>',
    @destination_credential = <CesCredential>,
    @max_message_size_kb    = 256,
    @partition_key_scheme   = N'Table';
GO

EXEC sys.sp_add_object_to_event_stream_group
    N'DocumentChanges',
    N'dbo.Documents';
GO
```

Microsoft Entra managed identity is preferred when the named platform supports it. Grant the identity the `Azure Event Hubs Data Sender` role at the narrowest practical scope.

### Delivery semantics

CES provides **at-least-once delivery**. A consumer must therefore be **idempotent**: processing the same event more than once must not corrupt state or generate duplicate business results.

A consumer can use an event ID or a business/change identifier to detect duplicates.

### CES advantages

- Push-based, near-real-time integration.
- No CDC-style relational change tables are written back into the database.
- Event Hubs and Fabric Eventstream scale independently.
- Multiple consumer groups can process the same stream.
- CloudEvents provides a standard envelope.

### CES limitations and exam facts

- CES is currently **preview**.
- It requires the **full recovery model**.
- It streams only changes made after CES starts; it does not seed existing table contents.
- It emits DML changes, not DDL schema-change events.
- It streams from a writable primary database.
- It cannot coexist with CDC, transactional replication, Fabric Mirrored Databases for SQL Server, or Azure Synapse Link in the same database.
- If disabled, it does not later backfill changes made while it was off.
- A column value larger than 1 MB is truncated to 1 MB before the event is formed.
- Since August 15, 2026, new Azure SQL Database and SQL database in Fabric stream groups use the `AzureEventHubs` destination type, which uses the Kafka protocol; older AMQP configuration is deprecated.

## 5. Change Tracking

### Definition

Change Tracking is a lightweight synchronization feature. It records **which rows changed**, identified by primary key, plus version and operation metadata. It does not preserve full historical row values.

```text
SQL row changes
      ↓
Internal change metadata
      ↓
Changed primary key + operation + version
      ↓
Join to current base table for current values
```

### When to choose Change Tracking

Use it when:

- the consumer only needs to know which rows changed;
- the consumer can read current values from the source table;
- lower overhead is more important than retaining every intermediate value;
- synchronizing an offline client, cache, search index, or embedding store;
- a periodic job can poll for changes by version.

### Enable Change Tracking

The table must have a primary key.

```sql
ALTER DATABASE [AiDatabase]
SET CHANGE_TRACKING = ON
(
    CHANGE_RETENTION = 2 DAYS,
    AUTO_CLEANUP = ON
);
GO

ALTER TABLE dbo.Documents
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = ON);
GO
```

`TRACK_COLUMNS_UPDATED = ON` stores extra metadata indicating which non-key columns were affected. Enable it only if the application needs that information.

### Retrieve changes

```sql
DECLARE @LastSyncVersion BIGINT = @SavedVersion;

SELECT
    CT.DocumentId,
    CT.SYS_CHANGE_VERSION,
    CT.SYS_CHANGE_OPERATION,
    CT.SYS_CHANGE_COLUMNS,
    D.Title,
    D.Body
FROM CHANGETABLE
(
    CHANGES dbo.Documents,
    @LastSyncVersion
) AS CT
LEFT JOIN dbo.Documents AS D
    ON D.DocumentId = CT.DocumentId;
```

The `LEFT JOIN` matters: a deleted row remains in the Change Tracking result, but no longer exists in the base table.

### Important functions

| Function | Purpose |
|---|---|
| `CHANGETABLE(CHANGES table, version)` | Returns keys and metadata for rows changed after the supplied version |
| `CHANGETABLE(VERSION table, (key), (value))` | Returns the latest tracked version for a particular row |
| `CHANGE_TRACKING_CURRENT_VERSION()` | Returns the current database change-tracking version |
| `CHANGE_TRACKING_MIN_VALID_VERSION(object_id)` | Returns the oldest version that can safely be used as a baseline |
| `CHANGE_TRACKING_IS_COLUMN_IN_MASK()` | Tests whether a column appears in the change-column mask |
| `WITH CHANGE_TRACKING_CONTEXT` | Associates application context with DML changes |

### Validate the saved baseline

Retention cleanup can remove metadata before a client returns. Always compare its saved version with the minimum valid version:

```sql
DECLARE @MinimumValidVersion BIGINT =
    CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID(N'dbo.Documents'));

IF @SavedVersion < @MinimumValidVersion
BEGIN
    -- The incremental baseline is too old.
    -- Perform a full reinitialization/synchronization.
END;
```

### Change Tracking behavior

- `SYS_CHANGE_OPERATION` is `I`, `U`, or `D`.
- Only one result is returned for each row changed since the supplied baseline.
- Multiple intermediate operations are consolidated.
- Current inserted or updated data must be retrieved from the base table.
- Deleted row values are not retained; only the key and change metadata remain.
- The consumer saves a synchronization version after completing a consistent sync.
- Use snapshot isolation or another documented consistency strategy to avoid races between reading changes, reading base rows, and saving the next version.

### Change Tracking advantages

- Lower overhead than CDC when only changed keys are needed.
- No triggers and no change-table schema mirroring.
- Good fit for intermittently connected and bidirectional sync clients.
- Supports SQL Server, Azure SQL Database, Azure SQL Managed Instance, and SQL database in Microsoft Fabric.

### Change Tracking limitations

- No complete row history.
- No before/after values.
- No record of every intermediate operation.
- Requires a primary key on each tracked table.
- A client that falls behind the retention window must reinitialize.
- Not a security audit.

## 6. Audit logging

### Definition

Auditing records selected database and server activity for security monitoring, investigations, accountability, and regulatory requirements.

Audit logging answers questions such as:

- Who logged in successfully or unsuccessfully?
- Who selected from or modified a sensitive table?
- Which statement or stored procedure was executed?
- When did the event occur?
- Which server, database, object, and principal were involved?

Audit logging is **not** designed to supply a complete row-change feed for ETL, synchronization, or embedding maintenance.

### SQL Server Audit architecture

```text
Auditable action
      ↓
Server or database audit specification
      ↓
Server audit object
      ↓
File, Windows Application log, or Windows Security log
```

| Component | Purpose |
|---|---|
| Server audit | Defines the target and audit-write behavior |
| Server audit specification | Selects server-level action groups |
| Database audit specification | Selects database-level actions or action groups |
| Audit target | Receives the records |

SQL Server Audit uses Extended Events infrastructure internally, but SQL Audit and a user-created Extended Events session solve different problems.

### SQL Server Audit example

```sql
USE master;
GO

CREATE SERVER AUDIT SensitiveDataAudit
TO FILE
(
    FILEPATH = N'D:\SqlAudit\',
    MAXSIZE = 1 GB,
    MAX_ROLLOVER_FILES = 20
)
WITH
(
    QUEUE_DELAY = 1000,
    ON_FAILURE = FAIL_OPERATION
);
GO

ALTER SERVER AUDIT SensitiveDataAudit
WITH (STATE = ON);
GO

USE AiDatabase;
GO

CREATE DATABASE AUDIT SPECIFICATION SensitiveTableSpecification
FOR SERVER AUDIT SensitiveDataAudit
    ADD (SELECT, INSERT, UPDATE, DELETE
         ON dbo.CustomerSecrets BY public)
WITH (STATE = ON);
GO
```

The example uses `public` only to demonstrate broad coverage. Production designs should select the narrowest appropriate actions, objects, and principals.

### SQL Server audit failure choices

| `ON_FAILURE` choice | Behavior if the audit target cannot be written |
|---|---|
| `CONTINUE` | Database operations continue; some audit records can be lost |
| `FAIL_OPERATION` | Operations that generate audited events fail; unrelated operations can continue |
| `SHUTDOWN` | SQL Server shuts down; requires appropriate permission to configure |

The default is `CONTINUE`. The correct choice depends on whether availability or audit completeness is the greater requirement.

### Read SQL Server audit files

```sql
SELECT
    event_time,
    action_id,
    succeeded,
    server_principal_name,
    database_name,
    schema_name,
    object_name,
    statement
FROM sys.fn_get_audit_file
(
    N'D:\SqlAudit\*.sqlaudit',
    DEFAULT,
    DEFAULT
)
ORDER BY event_time DESC;
```

### Azure SQL auditing

Azure SQL Database auditing can write selected database events to:

- Azure Storage;
- a Log Analytics workspace; and/or
- Azure Event Hubs.

A server-level Azure auditing policy applies to existing and newly created databases on that logical server. A database-level policy scopes auditing to a specific database. If both apply, take care to avoid unnecessary duplicate records.

The default Azure SQL auditing policy includes:

- `BATCH_COMPLETED_GROUP`;
- `SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP`;
- `FAILED_DATABASE_AUTHENTICATION_GROUP`.

### Audit destination selection

| Destination | Strong fit |
|---|---|
| Azure Storage | Long-term retention, archival, compliance evidence |
| Log Analytics | Interactive KQL queries, dashboards, alerts, Azure Monitor analysis |
| Event Hubs | Forwarding to a SIEM or external real-time processing system |
| SQL Server audit file | On-premises/VM SQL Server investigation and retention |
| Windows Security log | Central Windows security controls; requires extra configuration |

### Audit design practices

- Audit the actions required by the threat model or compliance policy; excessive collection increases cost and noise.
- Restrict access to audit configuration and audit targets.
- Place audit evidence in a protected target separate from ordinary application data when practical.
- Define retention and immutability policies.
- Alert on audit-policy changes and failures.
- Prefer managed identity over stored access keys for Azure targets when supported.
- Test failure behavior and confirm that expected events actually arrive.
- Remember that enabling auditing helps compliance but does not by itself guarantee compliance.

## 7. Ledger tables

### Definition

SQL ledger provides **tamper evidence** for sensitive relational data. It maintains cryptographically linked transaction information and row history so that an organization can later verify whether ledger data was altered outside the recorded history.

Ledger applies to:

- SQL Server 2022 and later;
- Azure SQL Database; and
- Azure SQL Managed Instance.

```text
Ledger-table transaction
          ↓
Row versions + transaction metadata
          ↓
Merkle-tree root and hash-linked database block
          ↓
Database digest stored outside the database
          ↓
Later verification detects inconsistent/tampered data
```

> [!IMPORTANT]
> Ledger is **tamper-evident**, not magically tamper-proof. A sufficiently privileged attacker might bypass database controls and alter files, but verification against a trusted external digest exposes the inconsistency.

### Ledger table types

| Type | Allowed DML | History table | Best fit |
|---|---|---|---|
| **Updatable ledger table** | `INSERT`, `UPDATE`, and `DELETE` | Yes; previous row versions are retained automatically | High-integrity system of record whose data must still change normally |
| **Append-only ledger table** | `INSERT` only | No separate history table is needed | Security events, financial journals, chain-of-custody records, or other immutable event data |

Both types create a ledger view that exposes ledger operations and transaction information.

### Updatable ledger table

An updatable ledger table is also system-versioned. Updates and deletes preserve the earlier row version in a history table.

```sql
CREATE SCHEMA Compliance;
GO

CREATE TABLE Compliance.AccountBalance
(
    AccountId INT NOT NULL PRIMARY KEY,
    Balance   DECIMAL(18, 2) NOT NULL
)
WITH
(
    SYSTEM_VERSIONING = ON
    (
        HISTORY_TABLE = Compliance.AccountBalanceHistory
    ),
    LEDGER = ON
);
GO
```

Use an updatable ledger table when normal updates and deletes are required but historical integrity must be verifiable.

### Append-only ledger table

An append-only ledger table rejects `UPDATE` and `DELETE` operations through the database API.

```sql
CREATE TABLE Compliance.SecurityEvents
(
    EventId      BIGINT IDENTITY PRIMARY KEY,
    EventTime    DATETIME2 NOT NULL,
    Principal    SYSNAME NOT NULL,
    EventDetails NVARCHAR(2000) NOT NULL
)
WITH
(
    LEDGER = ON (APPEND_ONLY = ON)
);
GO
```

Use append-only ledger for facts that should never be changed after insertion.

### Ledger views and transaction identity

The ledger view combines current and historical ledger information. Join it with `sys.database_ledger_transactions` to see the committing principal and time:

```sql
SELECT
    T.commit_time,
    T.principal_name,
    L.AccountId,
    L.Balance,
    L.ledger_operation_type_desc
FROM Compliance.AccountBalance_Ledger AS L
JOIN sys.database_ledger_transactions AS T
    ON T.transaction_id = L.ledger_transaction_id
ORDER BY T.commit_time DESC;
```

Microsoft recommends querying change history through the generated ledger view rather than querying the history table directly.

### Database ledger and digests

The database ledger logically uses blockchain and Merkle-tree structures:

1. Each ledger-table transaction contributes row hashes.
2. Transactions are summarized into a Merkle-tree root.
3. Database-ledger blocks are hash-linked to previous blocks.
4. The latest block hash is represented by a **database digest**.
5. Digests are stored outside the database in trusted, tamper-resistant storage.
6. Verification recomputes hashes and compares them with trusted digests.

Strong digest destinations include:

- Azure Blob immutable storage; and
- Azure Confidential Ledger.

If an attacker can alter both the database and its digest, the evidence loses trust. Protect the digest separately from database administrators and database storage.

### Verify ledger integrity

When automatic digest storage is configured, verification can use the registered digest locations:

```sql
DECLARE @DigestLocations NVARCHAR(MAX) =
(
    SELECT *
    FROM sys.database_ledger_digest_locations
    FOR JSON AUTO, INCLUDE_NULL_VALUES
);

EXEC sys.sp_verify_database_ledger_from_digest_storage
    @DigestLocations;
GO
```

Verification recomputes SHA-256-based ledger hashes. A mismatch indicates possible tampering. Verification can be resource-intensive, so schedule it according to risk and database size.

### Ledger versus audit logging

| Question | SQL Audit | Ledger |
|---|---|---|
| Who executed a selected action? | Primary purpose | Transaction principal is available for ledger changes |
| Which logins succeeded or failed? | Yes | No |
| Can I prove protected row history was not altered? | Audit records alone don't cryptographically attest table state | Yes, by verifying against trusted digests |
| Does it preserve previous row values? | Not as a row-history system | Yes for updatable ledger tables |
| Does it block updates and deletes? | No | Append-only ledger does |
| Main use | Activity/accountability evidence | Data-integrity/tamper evidence |

For high-assurance systems, SQL Audit and ledger are complementary: audit selected access and administrative actions, and use ledger to verify protected data integrity.

### Ledger limitations and responsibilities

- Ledger adds storage and processing overhead for history and hashing.
- An updatable ledger table cannot later be converted back into an ordinary table.
- Existing ordinary tables aren't converted in place; migrate data into newly created ledger tables.
- Old rows can't simply be deleted from append-only tables or ledger history tables.
- Append-only prevents normal updates/deletes but still requires digest verification to detect out-of-band tampering.
- Creating a ledger table requires the `ENABLE LEDGER` permission.
- Protect, retain, and periodically verify external digests.

## 8. Audit logging versus other logging features

| Requirement | Correct feature |
|---|---|
| Prove who accessed a sensitive table | SQL/Azure SQL Audit |
| Prove that protected row history wasn't altered | Ledger table plus trusted digests and verification |
| Diagnose a slow query or engine wait | Extended Events and performance monitoring |
| Record Azure resource creation or configuration changes | Azure Activity Log |
| Record an application's business workflow | Application logs/telemetry |
| Reconstruct retained database row changes | CDC, temporal tables, or updatable ledger, depending on integrity requirements |
| Identify rows that need synchronization | Change Tracking |
| Publish database changes to an event broker | CES |

> [!WARNING]
> An audit record may contain a statement and execution context, but auditing is not a reliable substitute for a row-version history feature.

## 9. Choosing an embedding-maintenance method

Embeddings become stale when their source content changes. DP-800 expects you to select a maintenance method that fits the required latency, detail, and architecture.

| Scenario | Recommended starting choice | Why |
|---|---|---|
| Nightly job needs IDs of changed documents | Change Tracking | Lightweight; retrieve current document values by key |
| Pipeline needs all retained changes and before/after values | CDC | Relational history over an LSN interval |
| Azure Function must react to changes in near real time | CES | Pushes events to Event Hubs or Fabric Eventstream |
| Security team needs evidence of who changed sensitive data | Audit logging | Captures principals and selected actions |
| Regulator requires cryptographic evidence that records weren't tampered with | Ledger table | Hash-linked history and trusted digests provide verifiable integrity |

### Example CES embedding flow

```text
Document UPDATE
      ↓
CES CloudEvent
      ↓
Event Hubs consumer / Azure Function
      ↓
Generate new embedding
      ↓
Update VECTOR column
```

### Example Change Tracking embedding flow

```text
Scheduled job saves version 500
      ↓
Later queries CHANGETABLE after version 500
      ↓
Reads current text for I/U keys
      ↓
Regenerates embeddings
      ↓
Removes vectors for D keys
      ↓
Saves the new synchronization version
```

### Reliability rules

- Make processing idempotent.
- Save a durable checkpoint: LSN, change-tracking version, or event identifier/checkpoint.
- Handle deletes as well as inserts and updates.
- Ensure retention is longer than the maximum consumer outage for CDC or Change Tracking.
- Route poison messages or repeated failures for investigation.
- Monitor lag between the source change and embedding refresh.
- Avoid regeneration loops when writing the new embedding back to the same tracked table; filter changes or mark embedding-only updates appropriately.

## 10. Key differences explained through one example

Suppose this statement runs:

```sql
UPDATE dbo.Documents
SET Body = N'New vector-search content'
WHERE DocumentId = 42;
```

### CDC sees

- a before-image row and an after-image row when all changes with old values are requested;
- operation codes `3` and `4`;
- captured column values and LSN metadata;
- data stored in a CDC change table until cleanup.

### CES sees

- an update CloudEvent sent to Event Hubs or Fabric Eventstream;
- previous and new values, subject to configuration/format and limits;
- at-least-once external delivery.

### Change Tracking sees

- primary key `42`;
- operation `U`;
- the change version and optional changed-column mask;
- no old body and no historical new body—the consumer joins to `dbo.Documents` for the current value.

### Audit logging sees

- the audited update action if the policy includes it;
- identity, time, success, database/object context, and potentially the statement;
- security evidence, not a CDC-compatible before/after row feed.

### An updatable ledger table sees

- the new current row plus the earlier row in its protected history table;
- transaction and sequence metadata exposed through the ledger view;
- the committing principal and time through `sys.database_ledger_transactions`;
- hash-linked evidence that can later be verified against a trusted database digest.

## 11. Common exam traps

1. **CDC and Change Tracking are not synonyms.** CDC retains change values; Change Tracking primarily retains changed keys and metadata.
2. **Change Tracking does not preserve intermediate values.** It reports one consolidated result per key relative to the supplied baseline.
3. **CES pushes; CDC and Change Tracking are polled.**
4. **CES is at least once.** Consumers must tolerate duplicates.
5. **Audit logging is about accountability and security.** It is not the normal source for ETL or embedding updates.
6. **A Change Tracking client can become too old.** Compare its checkpoint with `CHANGE_TRACKING_MIN_VALID_VERSION`.
7. **Retention must exceed consumer downtime.** This applies to CDC and Change Tracking.
8. **Change Tracking requires a primary key.**
9. **CDC update codes `3` and `4` are the before- and after-images.**
10. **CES does not stream existing rows when first enabled.** Perform a separate initial load.
11. **CES and CDC cannot coexist in the same database.**
12. **Azure SQL Database CDC uses an internal scheduler.** It does not depend on user-managed SQL Server Agent jobs.
13. **Audit targets differ by platform.** SQL Server can use file/Windows logs; Azure SQL commonly uses Storage, Log Analytics, or Event Hubs.
14. **Audit failure behavior is a business decision.** `CONTINUE` favors availability; `FAIL_OPERATION` or `SHUTDOWN` favors audit completeness.
15. **Extended Events is mainly for monitoring and troubleshooting.** SQL Audit is the purpose-built security-auditing feature.
16. **Ledger is tamper-evident, not a replacement for access control.** Continue to use permissions, encryption, and auditing.
17. **Updatable and append-only ledger tables are different.** Updatable ledger retains protected history; append-only permits inserts only.
18. **The external digest establishes trust.** Store digests where database administrators or attackers can't silently replace them.
19. **Ledger is not a change-delivery mechanism.** Use CDC, CES, or Change Tracking to drive embedding refreshes and integrations.

## 12. Memory aids

### Five-feature mnemonic

```text
CDC    = Changes stored in Database change tables
CES    = Changes sent as External Stream events
CT     = Changed keys Tracked
Audit  = Actor and Activity recorded
Ledger = Linked hashes prove data integrity
```

### Detail-level ladder

```text
Least row detail                                      Most row detail
Change Tracking ───────────────────────────────────────────── CDC
changed keys/metadata                         retained change values

CES is chosen for delivery architecture; Audit is chosen for accountability;
Ledger is chosen for cryptographically verifiable integrity.
```

### Checkpoint mnemonic

```text
CDC             → LSN
Change Tracking → version
CES             → event-stream consumer checkpoint / event ID
Audit           → target retention and query time range
Ledger          → trusted digest and verification range
```

## 13. Practice questions

### Question 1

A nightly process must find which document rows changed and then read their current text to regenerate embeddings. It does not need old values. Which feature has the lowest appropriate overhead?

A. SQL Audit  
B. Change Tracking  
C. CDC with all old values  
D. Extended Events

<details>
<summary>Answer</summary>

**B. Change Tracking.** It returns changed keys and operation/version metadata. The job can join to the base table to retrieve current text.

</details>

### Question 2

An ETL process needs every retained update, including before- and after-images. Which feature should it use?

A. Change Tracking  
B. SQL Audit  
C. CDC  
D. Dynamic Data Masking

<details>
<summary>Answer</summary>

**C. CDC.** The all-changes function can return update before- and after-images.

</details>

### Question 3

Several independent services must react to SQL changes through Azure Event Hubs with minimal latency. Which feature is designed for this?

A. Change Tracking  
B. CES  
C. SQL Audit file  
D. A full database backup

<details>
<summary>Answer</summary>

**B. CES.** It streams SQL changes to Azure Event Hubs or Fabric Eventstream for event-driven consumers.

</details>

### Question 4

A security administrator must determine who queried a sensitive salary table. Which feature is appropriate?

A. Change Tracking  
B. CDC  
C. SQL Audit  
D. A vector index

<details>
<summary>Answer</summary>

**C. SQL Audit.** Auditing records selected actions and execution identity/context for security investigation.

</details>

### Question 5

A Change Tracking client saved version 100. The table's minimum valid version is now 125. What should the client do?

A. Query changes from version 100 anyway  
B. Perform a full reinitialization  
C. Enable CDC automatically  
D. Read an audit file

<details>
<summary>Answer</summary>

**B.** Cleanup has removed metadata needed to synchronize safely from version 100, so the client must reinitialize.

</details>

### Question 6

Which Change Tracking construct returns changed row keys after a saved version?

A. `CHANGETABLE(CHANGES ...)`  
B. `sys.fn_get_audit_file`  
C. `VECTOR_SEARCH`  
D. `sp_cdc_enable_db`

<details>
<summary>Answer</summary>

**A. `CHANGETABLE(CHANGES ...)`.**

</details>

### Question 7

What does CDC operation code `3` represent?

A. Insert row  
B. Delete row  
C. Update before-image  
D. Update after-image

<details>
<summary>Answer</summary>

**C.** Code `3` is the update before-image; code `4` is the after-image.

</details>

### Question 8

A CES consumer occasionally receives the same event twice. What design characteristic is required?

A. The consumer must be idempotent.  
B. Disable all primary keys.  
C. Replace Event Hubs with an audit file.  
D. Ignore every second event.

<details>
<summary>Answer</summary>

**A.** CES provides at-least-once delivery, so a consumer must safely detect or tolerate duplicate processing.

</details>

### Question 9

Which feature requires a primary key on every tracked table?

A. Change Tracking  
B. SQL Server Audit  
C. CES  
D. Azure Activity Log

<details>
<summary>Answer</summary>

**A. Change Tracking.** It uses primary-key values to identify changed rows.

</details>

### Question 10

Which statement about Change Tracking is correct?

A. It preserves every intermediate row value.  
B. It stores full before/after images.  
C. It returns one consolidated change result per key relative to a baseline.  
D. It streams CloudEvents directly to Event Hubs.

<details>
<summary>Answer</summary>

**C.** Change Tracking is optimized to identify changed rows, not to preserve complete row history.

</details>

### Question 11

An Azure SQL team wants interactive KQL queries and alerts over security audit events. Which audit destination is the best fit?

A. Log Analytics workspace  
B. CDC change table  
C. Change Tracking internal table  
D. Vector column

<details>
<summary>Answer</summary>

**A. Log Analytics.** It integrates audit events with Azure Monitor queries, dashboards, and alerting.

</details>

### Question 12

What happens when SQL Server Audit uses `ON_FAILURE = CONTINUE` and cannot write to the target?

A. SQL Server always shuts down.  
B. Every database operation fails.  
C. Operations continue, and some audit records can be lost.  
D. CDC is enabled automatically.

<details>
<summary>Answer</summary>

**C.** `CONTINUE` prioritizes database availability over guaranteed audit completeness.

</details>

### Question 13

A team enables CES after a table already contains one million rows. How are those existing rows handled?

A. CES automatically emits one million insert events.  
B. CES streams only subsequent changes; the team needs a separate initial load.  
C. CES converts them into audit records.  
D. The table is truncated.

<details>
<summary>Answer</summary>

**B.** CES does not seed existing table contents when it is enabled.

</details>

### Question 14

Can CES be enabled in a database that already has CDC enabled?

A. Yes, always  
B. Yes, but only for the same tables  
C. No  
D. Only when auditing is disabled

<details>
<summary>Answer</summary>

**C.** Current CES limitations prohibit enabling it in a CDC-enabled database.

</details>

### Question 15

Which tool is the best fit for low-overhead engine performance troubleshooting rather than security auditing?

A. Extended Events  
B. CDC  
C. Change Tracking  
D. CES

<details>
<summary>Answer</summary>

**A. Extended Events.** It is the primary lightweight event framework for engine monitoring and performance troubleshooting.

</details>

### Question 16

A financial system must allow account balances to be updated while preserving cryptographically verifiable previous values. Which table type should it use?

A. Append-only ledger table  
B. Updatable ledger table  
C. Change Tracking table  
D. Temporary table

<details>
<summary>Answer</summary>

**B. Updatable ledger table.** It permits normal updates and deletes while preserving previous row versions in a protected history table.

</details>

### Question 17

A security event record must never be updated or deleted through the SQL API after insertion. Which table type is the best fit?

A. Updatable ledger table  
B. Append-only ledger table  
C. CDC change table  
D. Change Tracking table

<details>
<summary>Answer</summary>

**B. Append-only ledger table.** It accepts inserts and blocks updates and deletes.

</details>

### Question 18

Why should database ledger digests be stored outside the database in protected storage?

A. To make vector search faster  
B. To let consumers query Change Tracking  
C. To provide a trusted value for detecting database tampering  
D. To enable CDC net changes

<details>
<summary>Answer</summary>

**C.** Verification recomputes the ledger hashes and compares them with the externally protected digest. If an attacker could silently modify both, the verification evidence wouldn't be trustworthy.

</details>

## 14. Rapid-review checklist

- [ ] I can explain what CDC stores and how a consumer uses LSNs.
- [ ] I know CDC operation codes `1`, `2`, `3`, and `4`.
- [ ] I can distinguish CDC all-changes and net-changes queries.
- [ ] I know Azure SQL Database uses an internal CDC scheduler.
- [ ] I can explain why CDC retention must exceed consumer downtime.
- [ ] I know CES destinations and its CloudEvents format.
- [ ] I know CES is near-real-time, preview, and at-least-once.
- [ ] I know CES does not seed existing rows or emit DDL events.
- [ ] I know CES and CDC cannot coexist in the same database.
- [ ] I can configure Change Tracking at database and table scope.
- [ ] I know Change Tracking returns keys and metadata, not old values.
- [ ] I can use and validate a Change Tracking synchronization version.
- [ ] I know why a deleted Change Tracking result requires a `LEFT JOIN`.
- [ ] I can distinguish SQL Audit from Extended Events and Azure Activity Log.
- [ ] I know the SQL Server audit components and failure choices.
- [ ] I can choose among Azure Storage, Log Analytics, and Event Hubs audit targets.
- [ ] I can distinguish updatable from append-only ledger tables.
- [ ] I know ledger is tamper-evident and relies on protected external digests.
- [ ] I can explain how ledger verification detects inconsistent hashes.
- [ ] I can distinguish SQL Audit accountability from ledger data-integrity evidence.
- [ ] I can select CDC, CES, or Change Tracking for an embedding-maintenance scenario.

## 15. Official Microsoft references

- [DP-800 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [What is Change Data Capture?](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver17)
- [CDC with Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/change-data-capture-overview?view=azuresql)
- [Change Event Streaming overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview?view=sql-server-ver17)
- [Configure Change Event Streaming](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/configure?view=sql-server-ver17)
- [CES message format](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/message-format?view=sql-server-ver17)
- [CES frequently asked questions](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/frequently-asked-questions-faq?view=sql-server-ver17)
- [About Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-tracking-sql-server?view=sql-server-ver17)
- [Work with Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/work-with-change-tracking-sql-server?view=sql-server-ver17)
- [`CHANGETABLE`](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/changetable-transact-sql?view=sql-server-ver17)
- [SQL Server Audit](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine?view=sql-server-ver17)
- [Create a server audit and database audit specification](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/create-a-server-audit-and-database-audit-specification?view=sql-server-ver17)
- [Azure SQL Database auditing](https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview?view=azuresql)
- [Set up Azure SQL auditing](https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-setup?view=azuresql)
- [Extended Events overview](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/extended-events?view=sql-server-ver17)
- [Ledger overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-overview?view=sql-server-ver17)
- [Create an updatable ledger table](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-how-to-updatable-ledger-tables?view=sql-server-ver17)
- [Create an append-only ledger table](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-how-to-append-only-ledger-tables?view=sql-server-ver17)
- [Database ledger](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-database-ledger?view=sql-server-ver17)
- [Verify a ledger table to detect tampering](https://learn.microsoft.com/en-us/sql/relational-databases/security/ledger/ledger-verify-database?view=sql-server-ver17)

---

> **One-sentence exam summary:** Choose CDC for retained relational change values, CES for near-real-time external events, Change Tracking for lightweight changed-key synchronization, SQL Audit for activity accountability, and ledger tables for cryptographically verifiable, tamper-evident data history.
