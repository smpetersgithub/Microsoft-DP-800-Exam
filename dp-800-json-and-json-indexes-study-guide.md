# DP-800 Study Guide: JSON and JSON Indexes

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topic:** JSON storage, validation, querying, generation, modification, and indexing  
> **Last reviewed:** September 26, 2026

## Exam objectives covered

The current DP-800 study guide explicitly expects you to be able to:

- Design and implement JSON columns and indexes.
- Write queries that use JSON functions such as `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG`, `JSON_CONTAINS`, `OPENJSON`, and `JSON_VALUE`.

This guide also covers closely related functions and techniques that help you solve those objectives: `ISJSON`, `JSON_PATH_EXISTS`, `JSON_QUERY`, `JSON_MODIFY`, `JSON_OBJECTAGG`, `FOR JSON`, computed-column indexes, and native JSON indexes.

> [!IMPORTANT]
> JSON features differ by SQL Server version and Microsoft cloud platform. In an exam question, identify the named engine and version before choosing a data type, function, or index.

## 1. JSON in SQL Server: the big picture

JSON—JavaScript Object Notation—is a text-based format for representing objects, arrays, and scalar values.

```json
{
  "orderId": 1001,
  "customer": {
    "id": 42,
    "name": "Ada"
  },
  "status": "Processing",
  "tags": ["priority", "online"],
  "total": 149.95
}
```

SQL Server can combine relational columns with JSON documents:

```text
Relational columns                         JSON column
┌─────────┬────────────┬─────────────────────────────────────────┐
│ OrderId │ CreatedAt  │ Payload                                 │
├─────────┼────────────┼─────────────────────────────────────────┤
│ 1001    │ 2026-09-26 │ {"customer":...,"tags":...,"total":...} │
└─────────┴────────────┴─────────────────────────────────────────┘
```

This **hybrid design** is often preferable to either extreme:

- Put stable, frequently joined, filtered, constrained, or aggregated attributes in relational columns.
- Put sparse, optional, nested, or rapidly evolving attributes in JSON.
- Expose and index important JSON properties when the workload searches them frequently.

### When JSON is a good fit

- The data naturally contains nested objects or arrays.
- Different records can have different optional attributes.
- The schema evolves frequently.
- An application exchanges JSON and storing the original payload is useful.
- You still need SQL joins, transactions, security, and relational reporting.

### When relational columns are usually better

- A value is required for nearly every row.
- It is a primary or foreign key.
- It needs a strong data type, default, unique constraint, or frequent validation.
- Queries frequently join, group, sort, or filter by it.
- Independent updates to the value are common.

> [!TIP]
> “Flexible schema” does not mean “no design.” Decide which properties are required, how their types are validated, and which paths need indexes.

## 2. Storage choices

### Option A: character storage

SQL Server 2016 and later can store JSON text in `NVARCHAR` or `VARCHAR` and query it with JSON functions.

```sql
CREATE TABLE dbo.OrdersText
(
    OrderId  BIGINT PRIMARY KEY,
    Payload  NVARCHAR(MAX) NOT NULL,
    CONSTRAINT CK_OrdersText_Payload_IsJson
        CHECK (ISJSON(Payload, OBJECT) = 1)
);
```

Key points:

- `NVARCHAR(MAX)` is the traditional, broadly compatible choice.
- Character columns do not automatically guarantee valid JSON.
- Add an `ISJSON` check constraint when valid JSON is required.
- Existing JSON functions work with character storage.
- Important properties can be exposed through computed columns and indexed with normal B-tree indexes.

### Option B: native `JSON` data type

SQL Server 2025 provides a native `JSON` data type that stores documents in an optimized binary representation.

```sql
CREATE TABLE dbo.Orders
(
    OrderId    BIGINT PRIMARY KEY,
    CreatedAt  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    Payload    JSON NOT NULL
);
```

Advantages over character storage include:

- The document is already parsed, improving reads.
- Individual values can be modified more efficiently where possible.
- Storage is optimized for compression.
- Existing JSON functions continue to work.
- A native JSON index can be created on the column.

Important details:

- Native `JSON` accepts a JSON **object or array**; a top-level JSON scalar is not accepted as a stored document.
- A native `JSON` value is validated when it is assigned.
- The data type cannot be a normal B-tree index key, although it can be an included column.
- `JSON_VALUE(... RETURNING data_type)` can return a typed value when the input is native `JSON`.
- The native `modify` method is the preferred update mechanism for a native `JSON` column.

### Availability snapshot

| Feature | First relevant SQL Server version | Exam note |
|---|---:|---|
| Core JSON functions, `OPENJSON`, `FOR JSON` | SQL Server 2016 | `OPENJSON` normally requires compatibility level 130 or later. |
| Variable JSON paths | SQL Server 2017 | Earlier SQL Server versions require a literal path in affected functions. |
| `ISJSON` type constraint, `JSON_OBJECT`, `JSON_ARRAY`, `JSON_PATH_EXISTS` | SQL Server 2022 | Check the exact platform because cloud availability can differ. |
| Native `JSON`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`, `JSON_CONTAINS`, `CREATE JSON INDEX` | SQL Server 2025 | Major focus of the current DP-800 blueprint. |

> [!NOTE]
> Azure SQL Database, eligible Azure SQL Managed Instance configurations, and SQL database in Microsoft Fabric may receive features on different schedules. Treat the platform/version named in the question as a requirement.

## 3. Validate JSON

### `ISJSON`

`ISJSON` checks whether character data is valid JSON. It returns:

- `1` for valid JSON;
- `0` for invalid JSON; and
- `NULL` when the input is SQL `NULL`.

```sql
SELECT ISJSON(N'{"name":"Ada"}');          -- 1
SELECT ISJSON(N'not JSON');                 -- 0
SELECT ISJSON(N'[1,2,3]', ARRAY);           -- 1
SELECT ISJSON(N'{"x":1}', OBJECT);         -- 1
SELECT ISJSON(N'42', SCALAR);               -- 1
SELECT ISJSON(N'true', VALUE);              -- 1
```

The optional constraint, introduced in SQL Server 2022, can be:

| Constraint | What it accepts |
|---|---|
| `OBJECT` | A JSON object. |
| `ARRAY` | A JSON array. |
| `SCALAR` | A JSON number or string at the top level. |
| `VALUE` | An object, array, number, string, `true`, `false`, or `null`. |

Without the type constraint, `ISJSON` tests for an object or array.

> [!WARNING]
> `ISJSON` validates syntax, not your entire business schema. It does not check duplicate key names at the same level, require a particular property, or guarantee that a property has the desired SQL type.

### Validate document shape

Combine validation functions and constraints when key paths are required:

```sql
CREATE TABLE dbo.CustomerProfiles
(
    CustomerId  INT PRIMARY KEY,
    Profile      NVARCHAR(MAX) NOT NULL,

    CONSTRAINT CK_Profile_ValidObject
        CHECK (ISJSON(Profile, OBJECT) = 1),

    CONSTRAINT CK_Profile_HasEmail
        CHECK (JSON_PATH_EXISTS(Profile, '$.email') = 1)
);
```

For stronger typing, expose a property through a computed column or validate it in the ingestion procedure.

## 4. SQL/JSON path expressions

A JSON path identifies a location within a document.

```text
$                         root document
$.customer                customer object
$.customer.name           nested scalar property
$.tags[0]                 first array element
$[last]                   last array element (newer SQL/JSON syntax)
$.*                       child properties selected by a wildcard
$."property with spaces"  quoted key name
```

### Lax versus strict mode

```sql
SELECT JSON_VALUE(@doc, 'lax $.missing');    -- normally NULL
SELECT JSON_VALUE(@doc, 'strict $.missing'); -- error
```

| Mode | Behavior when a path cannot be resolved |
|---|---|
| `lax` | Usually returns `NULL` or an empty result. This is the default. |
| `strict` | Raises an error. |

Use `lax` for optional fields. Use `strict` when a missing or incompatible value represents bad data.

> [!IMPORTANT]
> Property-name matching in JSON paths is case-sensitive and collation-unaware in relevant path matching operations. `$.customerId` and `$.CustomerId` can refer to different keys.

### Array wildcards

SQL Server 2025 supports ANSI SQL/JSON array wildcards, ranges, lists, and `last` for native `JSON` input, for example:

```text
$.cards[*].type
$.cards[0 to 2].type
$.cards[0, 2].type
$.cards[last].type
```

`JSON_QUERY(... WITH ARRAY WRAPPER)` can wrap multiple selected values into a JSON array.

## 5. Read JSON values

### Function selection map

| Requirement | Best tool |
|---|---|
| Extract one scalar | `JSON_VALUE` |
| Extract an object or array | `JSON_QUERY` |
| Test whether a path exists | `JSON_PATH_EXISTS` |
| Turn an array/object into rows | `OPENJSON` |
| Search for a value inside JSON | `JSON_CONTAINS` |

### `JSON_VALUE`: return one scalar

```sql
DECLARE @doc NVARCHAR(MAX) = N'{
  "customer":{"id":42,"name":"Ada"},
  "tags":["priority","online"],
  "total":149.95
}';

SELECT
    JSON_VALUE(@doc, '$.customer.name') AS CustomerName,
    CAST(JSON_VALUE(@doc, '$.customer.id') AS INT) AS CustomerId,
    CAST(JSON_VALUE(@doc, '$.total') AS DECIMAL(10,2)) AS Total;
```

Remember:

- It returns a scalar, not an object or array.
- Without `RETURNING`, the result is `NVARCHAR(4000)`.
- A scalar longer than 4,000 characters returns `NULL` in lax mode or an error in strict mode; use `OPENJSON` for longer values.
- With native `JSON` in SQL Server 2025, use `RETURNING` for a typed result:

```sql
SELECT JSON_VALUE(Payload, '$.total' RETURNING DECIMAL(10,2))
FROM dbo.Orders;
```

### `JSON_QUERY`: return an object or array

```sql
SELECT
    JSON_QUERY(@doc, '$.customer') AS CustomerObject,
    JSON_QUERY(@doc, '$.tags') AS TagsArray;
```

Choosing the wrong extraction function is a common exam trap:

| Path result | `JSON_VALUE` | `JSON_QUERY` |
|---|---|---|
| String, number, Boolean scalar | Returns scalar text | `NULL` in lax mode |
| Object or array | `NULL` in lax mode | Returns JSON fragment |

With native `JSON` and array wildcard syntax:

```sql
SELECT JSON_QUERY(Payload, '$.items[*].sku' WITH ARRAY WRAPPER)
FROM dbo.Orders;
```

### `JSON_PATH_EXISTS`: test for a path

```sql
SELECT OrderId
FROM dbo.Orders
WHERE JSON_PATH_EXISTS(Payload, '$.shipping.trackingNumber') = 1;
```

It returns `1`, `0`, or `NULL` and does not raise an error for a missing path.

### `JSON_CONTAINS`: search within a document

`JSON_CONTAINS` is a SQL Server 2025 function that tests whether a scalar, object, or array occurs in the specified location.

```sql
SELECT CustomerId
FROM dbo.CustomerProfiles
WHERE JSON_CONTAINS(Profile, 'Azure', '$.skills[*]') = 1;
```

Typical uses include:

- searching an array for a value;
- checking whether an object contains a requested object fragment; and
- using a native JSON index to accelerate containment searches.

Do not confuse these two questions:

- “Does this **path** exist?” → `JSON_PATH_EXISTS`
- “Does this **value** occur in the document/path?” → `JSON_CONTAINS`

## 6. Convert JSON to rows with `OPENJSON`

`OPENJSON` is a table-valued function. It appears in the `FROM` clause and returns a rowset.

### Default schema

```sql
DECLARE @doc NVARCHAR(MAX) = N'{"name":"Ada","active":true,"score":98}';

SELECT [key], [value], [type]
FROM OPENJSON(@doc);
```

The default output contains:

| Column | Meaning |
|---|---|
| `key` | Object property name or zero-based array index. |
| `value` | Text representation of the value. |
| `type` | Integer code identifying null, string, number, Boolean, array, or object. |

### Explicit schema

An explicit `WITH` clause converts properties into typed SQL columns:

```sql
DECLARE @order NVARCHAR(MAX) = N'{
  "orderId":1001,
  "customer":{"id":42,"name":"Ada"},
  "items":[
    {"sku":"A-10","quantity":2,"unitPrice":19.95},
    {"sku":"B-20","quantity":1,"unitPrice":110.05}
  ]
}';

SELECT
    j.OrderId,
    j.CustomerId,
    j.CustomerName,
    j.Items
FROM OPENJSON(@order)
WITH
(
    OrderId       INT             '$.orderId',
    CustomerId    INT             '$.customer.id',
    CustomerName  NVARCHAR(100)   '$.customer.name',
    Items         NVARCHAR(MAX)   '$.items' AS JSON
) AS j;
```

Use `AS JSON` to preserve a nested object or array for a second parsing step.

### Expand an array with `CROSS APPLY`

```sql
SELECT
    o.OrderId,
    i.Sku,
    i.Quantity,
    i.UnitPrice
FROM dbo.Orders AS o
CROSS APPLY OPENJSON(o.Payload, '$.items')
WITH
(
    Sku        NVARCHAR(30)  '$.sku',
    Quantity   INT           '$.quantity',
    UnitPrice  DECIMAL(10,2) '$.unitPrice'
) AS i;
```

`CROSS APPLY` removes the parent row when the array produces no rows. Use `OUTER APPLY` when you need to preserve the parent row.

### Compatibility-level trap

`OPENJSON` normally requires database compatibility level 130 or later:

```sql
SELECT name, compatibility_level
FROM sys.databases
WHERE name = DB_NAME();
```

Other original JSON scalar functions are not subject to that same compatibility-level requirement.

## 7. Generate JSON

### `JSON_OBJECT`

Construct one object from key/value expressions:

```sql
SELECT JSON_OBJECT(
    'orderId': OrderId,
    'status': Status,
    'total': TotalAmount
) AS OrderJson
FROM dbo.OrderSummary;
```

`JSON_OBJECT` uses `NULL ON NULL` by default, so a SQL `NULL` normally becomes a JSON `null` property. Use `ABSENT ON NULL` to omit the property.

### `JSON_ARRAY`

Construct one array from expressions:

```sql
SELECT JSON_ARRAY('SQL', 'Azure', NULL ABSENT ON NULL);
-- ["SQL","Azure"]
```

`JSON_ARRAY` uses `ABSENT ON NULL` by default.

### `JSON_ARRAYAGG`

Aggregate values from multiple rows into one array:

```sql
SELECT
    CustomerId,
    JSON_ARRAYAGG(OrderId ORDER BY OrderId) AS OrderIds
FROM dbo.Orders
GROUP BY CustomerId;
```

This is different from `JSON_ARRAY`:

| Function | Input shape |
|---|---|
| `JSON_ARRAY(expr1, expr2, ...)` | Several expressions in the current row. |
| `JSON_ARRAYAGG(expr)` | One expression collected across multiple rows. |

### `JSON_OBJECTAGG`

Aggregate key/value pairs into an object:

```sql
SELECT JSON_OBJECTAGG(ProductCode : Quantity)
FROM dbo.OrderLines
WHERE OrderId = 1001;
```

### `FOR JSON`

`FOR JSON` serializes a query result:

```sql
SELECT
    o.OrderId AS 'order.id',
    o.Status AS 'order.status',
    o.TotalAmount AS 'order.total'
FROM dbo.OrderSummary AS o
FOR JSON PATH, ROOT('orders');
```

| Mode or option | Purpose |
|---|---|
| `FOR JSON AUTO` | Derives JSON nesting mainly from query/table structure. |
| `FOR JSON PATH` | Gives explicit control through column aliases and dot paths. |
| `ROOT('name')` | Adds a named root object. |
| `INCLUDE_NULL_VALUES` | Emits properties whose SQL values are `NULL`. |
| `WITHOUT_ARRAY_WRAPPER` | Removes the outer array; use only when a single-object result is appropriate. |

Use `JSON_QUERY` around a known JSON fragment to prevent `FOR JSON` from escaping it as an ordinary string.

## 8. Modify JSON

### `JSON_MODIFY`

`JSON_MODIFY` works with character JSON and returns updated JSON text.

```sql
DECLARE @profile NVARCHAR(MAX) =
    N'{"name":"Ada","skills":["SQL"],"city":"London"}';

-- Update an existing value.
SET @profile = JSON_MODIFY(@profile, '$.city', 'Chicago');

-- Insert a new property in lax mode.
SET @profile = JSON_MODIFY(@profile, '$.active', CAST(1 AS BIT));

-- Append to an array.
SET @profile = JSON_MODIFY(@profile, 'append $.skills', 'Azure');

-- Delete a property: lax mode plus SQL NULL.
SET @profile = JSON_MODIFY(@profile, 'lax $.city', NULL);

-- Store JSON null instead of deleting: strict mode plus SQL NULL.
SET @profile = JSON_MODIFY(@profile, 'strict $.name', NULL);
```

Important rules:

- One `JSON_MODIFY` call changes one path; nest calls for multiple changes.
- `append` adds an element to an array.
- In lax mode, setting an existing property to SQL `NULL` deletes it.
- In strict mode, setting an existing property to SQL `NULL` stores JSON `null`.
- Cast numbers and Booleans to the correct SQL type, or they may be stored as quoted strings.
- Wrap a nested JSON value in `JSON_QUERY` to avoid double escaping.

### Native `JSON.modify()`

For a native `JSON` column, use its `modify` method:

```sql
UPDATE dbo.Orders
SET Payload.modify('$.status', 'Shipped')
WHERE OrderId = 1001;
```

This method can perform in-place modification when possible and is preferred for native `JSON` documents.

## 9. JSON indexes

There are two important patterns:

1. A normal B-tree index on a computed column extracted with `JSON_VALUE`.
2. A native JSON index created with `CREATE JSON INDEX`.

### Pattern A: computed-column B-tree index

This pattern works with JSON stored in character columns and is supported from SQL Server 2016 onward. It can also be used with native `JSON`.

```sql
CREATE TABLE dbo.OrdersText
(
    OrderId  BIGINT PRIMARY KEY,
    Payload  NVARCHAR(MAX) NOT NULL,
    CONSTRAINT CK_OrdersText_Payload
        CHECK (ISJSON(Payload, OBJECT) = 1)
);

ALTER TABLE dbo.OrdersText
ADD CustomerId AS
    CAST(JSON_VALUE(Payload, '$.customer.id') AS INT);

CREATE INDEX IX_OrdersText_CustomerId
    ON dbo.OrdersText(CustomerId)
    INCLUDE (OrderId);
```

A query can use the computed-column index when the JSON expression matches:

```sql
SELECT OrderId
FROM dbo.OrdersText
WHERE CAST(JSON_VALUE(Payload, '$.customer.id') AS INT) = 42;
```

SQL Server can recognize that the expression is equivalent to the indexed computed column; the query does not necessarily have to reference `CustomerId` by name.

#### Computed-column index rules to remember

- The path and expression in the query should match the computed-column definition.
- Cast the extracted value to the smallest appropriate type.
- `JSON_VALUE` without a cast can return `NVARCHAR(4000)`, whose potential size exceeds the 1,700-byte nonclustered index key limit.
- If an indexed string value exceeds the key limit, a later insert or update can fail.
- Numeric, date, and Boolean-like properties should normally be converted to appropriate narrow SQL types.
- The computed column does not have to be `PERSISTED` to be indexed when the expression meets computed-column index requirements.
- `INCLUDE` relational columns when doing so avoids key lookups for an important query.
- The result of `JSON_VALUE` inherits the source expression's collation, so string ordering and comparison are collation-aware.

Example with a bounded string:

```sql
ALTER TABLE dbo.OrdersText
ADD StatusCode AS
    CAST(JSON_VALUE(Payload, '$.status') AS NVARCHAR(30));

CREATE INDEX IX_OrdersText_StatusCode
    ON dbo.OrdersText(StatusCode);
```

### Pattern B: native JSON index

SQL Server 2025 can build a JSON-aware index on a native `JSON` column.

```sql
CREATE TABLE dbo.Orders
(
    OrderId  BIGINT PRIMARY KEY CLUSTERED,
    Payload  JSON NOT NULL
);

-- Index the entire document recursively from the root.
CREATE JSON INDEX IX_Orders_Payload
    ON dbo.Orders(Payload);
```

The default path is `$`, which recursively indexes keys and values in the document.

You can instead target selected, nonoverlapping paths:

```sql
CREATE JSON INDEX IX_Orders_Payload
    ON dbo.Orders(Payload)
    FOR ('$.status', '$.customer.id');
```

For array-oriented searches:

```sql
CREATE JSON INDEX IX_Profiles_Profile
    ON dbo.CustomerProfiles(Profile)
    WITH (OPTIMIZE_FOR_ARRAY_SEARCH = ON);
```

> [!NOTE]
> The final example requires `Profile` to be a native `JSON` column, not `NVARCHAR(MAX)`.

#### What a native JSON index can accelerate

The optimizer can use the index for supported predicates involving:

- `JSON_VALUE` comparisons;
- `JSON_PATH_EXISTS`; and
- `JSON_CONTAINS`.

Examples:

```sql
SELECT OrderId
FROM dbo.Orders
WHERE JSON_VALUE(Payload, '$.status') = 'Shipped';

SELECT OrderId
FROM dbo.Orders
WHERE JSON_VALUE(Payload, '$.customer.id' RETURNING INT) = 42;

SELECT OrderId
FROM dbo.Orders
WHERE JSON_PATH_EXISTS(Payload, '$.shipping.trackingNumber') = 1;

SELECT CustomerId
FROM dbo.CustomerProfiles
WHERE JSON_CONTAINS(Profile, 'Azure', '$.skills[*]') = 1;
```

#### Native JSON index requirements and limitations

- The column must use the native `JSON` data type.
- The table must have a clustered primary key.
- Only one JSON index can exist on a particular JSON column.
- Explicitly indexed paths cannot overlap. For example, `$.customer` already recursively includes `$.customer.id`, so listing both is invalid.
- A JSON index cannot be created on an indexed view, computed JSON column, memory-optimized table, or view.
- JSON index creation, rebuild, and drop are offline operations and take a schema-modification lock.
- Changing the indexed paths requires recreating the JSON index.
- Current documented predicate limitations include unsupported `LIKE` and `IS NULL` optimization for `JSON_VALUE`.
- Every index adds storage and write-maintenance cost; index for the actual workload.

### Computed-column index versus native JSON index

| Requirement | Computed-column B-tree index | Native JSON index |
|---|---|---|
| Earliest SQL Server version | 2016 | 2025 |
| Source storage | `VARCHAR`, `NVARCHAR`, or native `JSON` | Native `JSON` only |
| Number of properties | Usually one computed expression per key column | Many paths can be indexed recursively |
| Best for | Known, stable scalar paths and ordinary relational index design | Broader JSON path, existence, and containment searches |
| Typed key control | Explicit cast in computed expression | Use JSON-aware index and typed extraction in queries |
| Included relational columns | Supported with a normal nonclustered index | Not the main design pattern |
| Clustered primary key required | No special JSON requirement | Yes |
| Array-search option | No dedicated option | `OPTIMIZE_FOR_ARRAY_SEARCH = ON` |

### Index-selection decision

```text
Does the question specify SQL Server 2025 + native JSON?
├─ No  → Use a computed column + normal B-tree index.
└─ Yes
   ├─ One stable scalar path and a tailored covering index? → Computed column may fit.
   └─ Multiple/dynamic paths, existence, or containment?    → Consider CREATE JSON INDEX.
```

## 10. End-to-end design example

This design keeps frequently queried values relational while retaining flexible details in JSON.

```sql
CREATE TABLE dbo.AIRequests
(
    RequestId     BIGINT IDENTITY PRIMARY KEY CLUSTERED,
    TenantId      INT NOT NULL,
    CreatedAt     DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    RequestStatus VARCHAR(20) NOT NULL,
    Metadata      JSON NOT NULL,

    CONSTRAINT CK_AIRequests_Metadata_Model
        CHECK (JSON_PATH_EXISTS(Metadata, '$.model') = 1)
);

CREATE INDEX IX_AIRequests_Tenant_Status_Created
    ON dbo.AIRequests(TenantId, RequestStatus, CreatedAt DESC);

CREATE JSON INDEX IX_AIRequests_Metadata
    ON dbo.AIRequests(Metadata)
    FOR ('$.model', '$.tags');
```

Why this is sensible:

- `TenantId`, status, and timestamp are stable and frequently filtered, so they remain relational.
- Optional model settings, tags, and trace metadata can evolve in JSON.
- A check constraint requires a model property.
- The relational index supports the dominant tenant/status/time query.
- The JSON index supports searches inside selected metadata paths.

## 11. Performance and security checklist

### Performance

- Store frequently searched business keys as relational columns when practical.
- Use native `JSON` for SQL Server 2025 workloads that benefit from parsed binary storage and JSON indexes.
- With text storage, validate input once and index important scalar paths through computed columns.
- Use an explicit `OPENJSON ... WITH` schema to return typed columns.
- Expand arrays with `CROSS APPLY` rather than repeatedly reparsing the same fragment in many scalar calls.
- Match the computed-column expression exactly in queries.
- Cast indexed values to narrow, appropriate SQL types.
- Use actual execution plans to verify an index seek; the presence of an index does not guarantee its use.
- Avoid indexing every possible path. Indexes increase storage, logging, and DML cost.

### Security and correctness

- Parameterize SQL values. Do not concatenate untrusted input into dynamic SQL.
- Whitelist JSON paths if an application allows users to choose paths.
- Apply normal SQL permissions, Row-Level Security, encryption, and auditing to tables containing JSON.
- Do not assume that nested JSON fields are automatically masked or encrypted.
- Validate both JSON syntax and required business structure.
- Avoid placing secrets in loosely governed payloads or AI prompts.

## 12. Common exam traps

1. **Using `JSON_VALUE` for an object or array.** Use `JSON_QUERY`.
2. **Using `JSON_QUERY` for a scalar.** Use `JSON_VALUE`.
3. **Treating `OPENJSON` as a scalar function.** It is a table-valued function used in `FROM` or `APPLY`.
4. **Forgetting compatibility level 130 for `OPENJSON`.** This is a classic troubleshooting clue.
5. **Assuming text columns validate JSON automatically.** Add an `ISJSON` check constraint.
6. **Assuming `ISJSON` enforces a schema.** It validates syntax, not all required properties and types.
7. **Forgetting that lax mode is the default.** Missing or mismatched paths often produce `NULL` instead of errors.
8. **Using `JSON_MODIFY(..., NULL)` without considering path mode.** Lax mode deletes; strict mode stores JSON `null` when the path exists.
9. **Using `JSON_ARRAY` when rows must be aggregated.** Use `JSON_ARRAYAGG` across rows.
10. **Indexing raw `NVARCHAR(MAX)` JSON directly.** Extract the scalar to a computed column and index that column.
11. **Leaving an indexed `JSON_VALUE` expression as `NVARCHAR(4000)`.** Cast it to a narrow type to avoid index-key-size problems.
12. **Choosing `CREATE JSON INDEX` for `NVARCHAR` JSON.** A native JSON index requires a native `JSON` column.
13. **Creating a native JSON index without a clustered primary key.** The table must have one.
14. **Specifying overlapping native JSON index paths.** A parent path already recursively includes its descendants.
15. **Assuming all JSON features exist in SQL Server 2022.** Native `JSON`, `JSON_CONTAINS`, `JSON_ARRAYAGG`, and `CREATE JSON INDEX` are SQL Server 2025-era features.

## 13. Quick-reference cheat sheet

```text
Validate JSON text                  ISJSON(document)
Require object/array/value          ISJSON(document, OBJECT|ARRAY|VALUE|SCALAR)
Extract one scalar                  JSON_VALUE(document, path)
Extract object or array             JSON_QUERY(document, path)
Check for a path                    JSON_PATH_EXISTS(document, path)
Search for a value                  JSON_CONTAINS(document, value, path)
Convert JSON to rows                OPENJSON(document) [WITH (...)]
Update JSON text                    JSON_MODIFY(document, path, newValue)
Update native JSON                  jsonColumn.modify(path, newValue)
Construct one object                JSON_OBJECT('key': value, ...)
Construct one array                 JSON_ARRAY(value, ...)
Aggregate rows into array           JSON_ARRAYAGG(value ORDER BY ...)
Aggregate rows into object          JSON_OBJECTAGG(key : value)
Serialize query result              SELECT ... FOR JSON PATH
Index one known scalar              computed column + CREATE INDEX
Index native JSON paths             CREATE JSON INDEX
```

### Memorize the six blueprint functions

| Function | One-sentence purpose |
|---|---|
| `JSON_OBJECT` | Builds one JSON object from key/value expressions. |
| `JSON_ARRAY` | Builds one JSON array from expressions in the current row. |
| `JSON_ARRAYAGG` | Aggregates values from multiple rows into a JSON array. |
| `JSON_CONTAINS` | Tests whether a value occurs in a JSON document or path. |
| `OPENJSON` | Converts an object or array into a relational rowset. |
| `JSON_VALUE` | Extracts one scalar value from a path. |

## 14. Practice questions

### Question 1

An `NVARCHAR(MAX)` column must reject malformed JSON objects. What should you add?

A. A default constraint  
B. `CHECK (ISJSON(Payload, OBJECT) = 1)`  
C. A foreign key  
D. `CHECK (JSON_VALUE(Payload, '$') = 1)`

<details>
<summary>Answer</summary>

**B.** `ISJSON` validates JSON syntax, and `OBJECT` requires the top-level value to be an object.

</details>

### Question 2

You need the scalar value at `$.customer.name`. Which function should you use?

A. `JSON_QUERY`  
B. `OPENJSON`  
C. `JSON_VALUE`  
D. `JSON_ARRAYAGG`

<details>
<summary>Answer</summary>

**C.** `JSON_VALUE` extracts one scalar.

</details>

### Question 3

You need the complete array at `$.items` as a JSON fragment. Which function should you use?

A. `JSON_VALUE`  
B. `JSON_QUERY`  
C. `ISJSON`  
D. `JSON_OBJECT`

<details>
<summary>Answer</summary>

**B.** `JSON_QUERY` returns an object or array.

</details>

### Question 4

You need one relational row for every object in `$.items`. Which design is best?

A. `JSON_VALUE` in the `ORDER BY` clause  
B. `CROSS APPLY OPENJSON(Payload, '$.items') WITH (...)`  
C. `JSON_QUERY(Payload, '$.items')` only  
D. `ISJSON(Payload)`

<details>
<summary>Answer</summary>

**B.** `OPENJSON` returns a rowset, and `CROSS APPLY` correlates array elements with the parent row.

</details>

### Question 5

`OPENJSON` is not recognized in an older database, but `JSON_VALUE` works. What should you check first?

A. Whether the database compatibility level is at least 130  
B. Whether the table has a clustered columnstore index  
C. Whether CLR is enabled  
D. Whether snapshot isolation is enabled

<details>
<summary>Answer</summary>

**A.** `OPENJSON` normally requires compatibility level 130 or later.

</details>

### Question 6

You need an array of `OrderId` values across all orders for each customer. Which function is the direct fit?

A. `JSON_ARRAY`  
B. `JSON_ARRAYAGG`  
C. `JSON_VALUE`  
D. `JSON_MODIFY`

<details>
<summary>Answer</summary>

**B.** `JSON_ARRAYAGG` aggregates values across rows. `JSON_ARRAY` combines expressions supplied to a single call.

</details>

### Question 7

What does this do when `$.city` exists?

```sql
SET @doc = JSON_MODIFY(@doc, 'lax $.city', NULL);
```

A. Stores the string `"NULL"`  
B. Stores JSON `null`  
C. Deletes the `city` property  
D. Raises an error

<details>
<summary>Answer</summary>

**C.** In lax mode, assigning SQL `NULL` deletes the property.

</details>

### Question 8

An application on SQL Server 2019 frequently filters an `NVARCHAR(MAX)` JSON column by `$.customer.id`. What is the best indexing pattern?

A. Create a JSON index directly on the text column  
B. Add a typed computed column based on `JSON_VALUE` and create a normal index on it  
C. Add a full-text index  
D. Index the first 8,000 characters of the JSON document

<details>
<summary>Answer</summary>

**B.** Computed-column B-tree indexing is the compatible JSON-property index pattern for that version.

</details>

### Question 9

Why should an indexed `JSON_VALUE` expression usually be cast to a narrow type?

A. `JSON_VALUE` otherwise returns XML  
B. JSON paths work only with integers  
C. Its default `NVARCHAR(4000)` result can exceed the nonclustered index-key limit  
D. Native JSON indexes require `CHAR(1)`

<details>
<summary>Answer</summary>

**C.** A narrow appropriate type avoids oversized key warnings and potential DML failures.

</details>

### Question 10

Which combination is required for `CREATE JSON INDEX` in SQL Server 2025?

A. An XML column and heap table  
B. An `NVARCHAR(MAX)` column and unique constraint  
C. A native `JSON` column and a clustered primary key on the table  
D. A persisted computed column and memory-optimized table

<details>
<summary>Answer</summary>

**C.** The native index targets a `JSON` column, and its table requires a clustered primary key.

</details>

### Question 11

Which native JSON index path list is invalid?

A. `FOR ('$.status', '$.customer.id')`  
B. `FOR ('$.customer', '$.customer.id')`  
C. `FOR ('$.model', '$.tags')`  
D. `FOR ('$.shipping.city', '$.billing.city')`

<details>
<summary>Answer</summary>

**B.** The paths overlap because recursively indexing `$.customer` already includes `$.customer.id`.

</details>

### Question 12

You need to know whether the path `$.shipping.trackingNumber` exists, regardless of its value. Which function is most direct?

A. `JSON_PATH_EXISTS`  
B. `JSON_ARRAY`  
C. `JSON_OBJECT`  
D. `JSON_MODIFY`

<details>
<summary>Answer</summary>

**A.** `JSON_PATH_EXISTS` tests whether the path resolves to a nonempty sequence.

</details>

### Question 13

You must find profiles whose `skills` array contains `Azure`. Which SQL Server 2025 function is designed for that search?

A. `ISJSON`  
B. `JSON_CONTAINS`  
C. `JSON_QUERY`  
D. `JSON_OBJECTAGG`

<details>
<summary>Answer</summary>

**B.** `JSON_CONTAINS` searches for a value within a JSON document or path.

</details>

### Question 14

You want to preserve parent table rows even when their JSON array is absent or empty. Which relational operator should you prefer with `OPENJSON`?

A. `CROSS JOIN`  
B. `INNER JOIN` without a predicate  
C. `OUTER APPLY`  
D. `INTERSECT`

<details>
<summary>Answer</summary>

**C.** `OUTER APPLY` preserves the parent row when the table-valued function produces no rows.

</details>

### Question 15

Which statement about lax and strict modes is correct?

A. Strict is the default and always returns `NULL`  
B. Lax is the default and normally tolerates a missing path  
C. Lax validates duplicate property names  
D. Strict works only with `OPENJSON`

<details>
<summary>Answer</summary>

**B.** Lax is the default; strict raises an error for path-resolution problems.

</details>

## 15. Official references

- [DP-800 official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [JSON data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)
- [Native JSON data type](https://learn.microsoft.com/en-us/sql/t-sql/data-types/json-data-type?view=sql-server-ver17)
- [JSON functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/json-functions-transact-sql?view=sql-server-ver17)
- [JSON path expressions](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-path-expressions-sql-server?view=sql-server-ver17)
- [OPENJSON](https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql?view=sql-server-ver17)
- [Index JSON data with computed columns](https://learn.microsoft.com/en-us/sql/relational-databases/json/index-json-data?view=sql-server-ver17)
- [CREATE JSON INDEX](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-json-index-transact-sql?view=sql-server-ver17)
- [JSON_CONTAINS](https://learn.microsoft.com/en-us/sql/t-sql/functions/json-contains-transact-sql?view=sql-server-ver17)

---

## Final memory aid

> **Scalar → `JSON_VALUE`; object/array → `JSON_QUERY`; rows → `OPENJSON`; path → `JSON_PATH_EXISTS`; contained value → `JSON_CONTAINS`; text property index → computed column; native JSON search index → `CREATE JSON INDEX`.**
