# DP-800 Study Guide: SQL Security, Data Masking, and Secure Enclaves

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topic:** Data protection, access control, auditing, identity, secure enclaves, and endpoint security  
> **Last reviewed:** September 26, 2026

## Exam objectives covered

The current DP-800 blueprint expects you to be able to:

- Design and implement data encryption, including Always Encrypted and column-level encryption.
- Design and implement Dynamic Data Masking.
- Design and implement Row-Level Security.
- Design and implement object-level permissions.
- Implement secure database access, including passwordless access.
- Implement auditing.
- Secure model endpoints, including with managed identity.
- Secure GraphQL, REST, and Model Context Protocol endpoints.
- Interpret the security impact of AI-assisted tools.

> [!IMPORTANT]
> Security features and syntax differ across SQL Server, Azure SQL Database, Azure SQL Managed Instance, Azure Synapse Analytics, and Microsoft Fabric. Always identify the platform and version named in an exam question.

## 1. Start with the security objective

Different controls solve different security problems.

| Requirement | Primary control |
|---|---|
| Prove who is connecting | Authentication |
| Control what an identity can do | Authorization and permissions |
| Restrict users to their own rows | Row-Level Security |
| Hide selected output values from nonprivileged users | Dynamic Data Masking |
| Protect database files and backups if storage is stolen | Transparent Data Encryption |
| Protect sensitive columns from database operators | Always Encrypted |
| Query encrypted data with richer predicates | Always Encrypted with secure enclaves |
| Protect data moving over a network | TLS |
| Record who did what | SQL auditing |
| Eliminate application passwords | Microsoft Entra ID and managed identity |
| Protect an API exposed over REST, GraphQL, or MCP | Authentication, authorization, transport security, and least privilege |

### Defense in depth

```text
Network boundary
    ↓ private endpoint / firewall / TLS
Identity
    ↓ Microsoft Entra ID / managed identity / MFA
Authorization
    ↓ least privilege / roles / GRANT / DENY
Data filtering
    ↓ Row-Level Security / field permissions
Data presentation
    ↓ Dynamic Data Masking
Data protection
    ↓ TDE / Always Encrypted / column-level encryption
Detection and evidence
    ↓ auditing / monitoring / alerts
```

No single layer replaces all the others.

## 2. Security concepts to distinguish

### Authentication versus authorization

- **Authentication** answers: “Who are you?”
- **Authorization** answers: “What are you allowed to do?”

A managed identity authenticates a workload without an application-managed secret. It still needs authorization in both Microsoft Entra/Azure and the database.

### Encryption versus masking

- **Encryption** transforms data into ciphertext and requires a key to recover plaintext.
- **Masking** changes how query results are displayed to selected users; the stored value remains unchanged.

### Prevention versus detection

- Permissions, encryption, RLS, and network rules help **prevent** unauthorized access.
- Auditing and monitoring help **detect, investigate, and prove** activity.

Auditing does not automatically block a prohibited action. Encryption does not decide whether a user should have access. Masking is not encryption.

## 3. Encryption selection guide

| Technology | Protects | Transparent to application? | Can a privileged database operator normally see plaintext? | Typical use |
|---|---|---:|---:|---|
| TLS | Data in transit | Usually | Yes, after the data reaches the engine | Protect network connections. |
| TDE | Data and log files at rest; TDE-protected backups | Yes | Yes | Protect stolen files, disks, and backups. |
| Backup encryption | Backup media | Yes during normal database access | Yes | Protect backup files independently of TDE. |
| Column/cell-level encryption | Selected values stored as ciphertext | No | Potentially, if allowed to open the key | T-SQL-controlled encryption of selected cells. |
| Always Encrypted | Selected columns in use, at rest, and in transit between client and engine | Requires an enabled client driver | No, by design | Protect sensitive data from DBAs and host operators. |
| Always Encrypted with secure enclaves | Same columns, plus protected in-engine computation | Requires enclave-capable setup and driver | Plaintext is limited to protected enclave memory | Rich confidential queries and in-place cryptographic operations. |

## 4. TLS: protect data in transit

TLS encrypts traffic between the client and SQL endpoint.

For client connections:

- Require encryption where supported.
- Validate the server certificate.
- Avoid `TrustServerCertificate=True` in production because it bypasses certificate-chain validation.
- Keep drivers and TLS versions current.
- Use private connectivity when appropriate, but remember that a private network does not replace TLS or identity controls.

Example connection-string intent:

```text
Server=tcp:<server>.database.windows.net,1433;
Database=<database>;
Authentication=Active Directory Managed Identity;
Encrypt=True;
TrustServerCertificate=False;
```

## 5. Transparent Data Encryption

Transparent Data Encryption (TDE) performs real-time encryption and decryption of database data and log files.

### What TDE protects

- Data files at rest.
- Transaction log files at rest.
- Backups of a TDE-enabled database.
- Stolen disks or backup media from being attached or restored without the protecting key or certificate.

### What TDE does not protect

- Data returned to an authorized query.
- Data from a compromised account that already has permission.
- Data from privileged operators while the database is online.
- Network traffic; use TLS for that.
- Individual columns from other authorized database users.

### SQL Server key hierarchy for TDE

```text
Service Master Key
    ↓ protects
Database Master Key in master
    ↓ protects
Server certificate or asymmetric key
    ↓ protects
Database Encryption Key in user database
    ↓ encrypts
Data and log files
```

### SQL Server example

```sql
USE master;
GO

CREATE MASTER KEY
    ENCRYPTION BY PASSWORD = '<strong-master-key-password>';
GO

CREATE CERTIFICATE TdeServerCertificate
    WITH SUBJECT = 'Certificate protecting the application DEK';
GO

USE ApplicationDb;
GO

CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TdeServerCertificate;
GO

ALTER DATABASE ApplicationDb SET ENCRYPTION ON;
GO
```

Check status:

```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    encryption_state,
    percent_complete,
    key_algorithm,
    key_length
FROM sys.dm_database_encryption_keys;
```

> [!CAUTION]
> Back up the certificate and its private key separately and protect the backup password. A TDE backup cannot be restored to another SQL Server instance without the certificate or key that protects its database encryption key.

In Azure SQL, TDE is platform-integrated. Depending on the service and configuration, keys can be service-managed or customer-managed in Azure Key Vault or Managed HSM.

## 6. Column-level or cell-level encryption

SQL Server can encrypt selected values by using a database master key, certificate, symmetric key, and T-SQL cryptographic functions.

```text
Database master key
    ↓ protects
Certificate
    ↓ protects
Symmetric key
    ↓ encrypts/decrypts
Selected column values
```

### Example pattern

```sql
CREATE MASTER KEY
    ENCRYPTION BY PASSWORD = '<strong-master-key-password>';
GO

CREATE CERTIFICATE CustomerDataCertificate
    WITH SUBJECT = 'Protect customer identifiers';
GO

CREATE SYMMETRIC KEY CustomerDataKey
    WITH ALGORITHM = AES_256
    ENCRYPTION BY CERTIFICATE CustomerDataCertificate;
GO

ALTER TABLE dbo.Customers
ADD GovernmentIdEncrypted VARBINARY(256) NULL;
GO

OPEN SYMMETRIC KEY CustomerDataKey
    DECRYPTION BY CERTIFICATE CustomerDataCertificate;

UPDATE dbo.Customers
SET GovernmentIdEncrypted = EncryptByKey(
    Key_GUID('CustomerDataKey'),
    GovernmentId
);

CLOSE SYMMETRIC KEY CustomerDataKey;
GO
```

To decrypt, an authorized session opens the key and calls `DecryptByKey`.

### Characteristics

- Encryption and decryption occur inside the Database Engine.
- The application or stored procedures must explicitly call cryptographic functions.
- The encrypted value is normally stored in a `VARBINARY` column.
- Key permissions determine who can open the key.
- Keys and certificates require backup and rotation plans.
- An authenticator can help prevent ciphertext from being moved from one row or context to another.

### Column-level encryption versus Always Encrypted

| Requirement | Column/cell-level encryption | Always Encrypted |
|---|---|---|
| Engine can access plaintext | Yes, when the key is open | No, except within a configured secure enclave |
| Protect from DBA/operator | Not the main guarantee | Yes |
| Encryption location | Database Engine | Always Encrypted-enabled client driver |
| Application/query changes | Explicit cryptographic functions | Driver integration and parameterization |
| Key hierarchy | SQL master key, certificate, symmetric key | Column master key and column encryption key |

## 7. Always Encrypted

Always Encrypted protects sensitive columns from high-privileged but unauthorized users, including database administrators and host operators.

### Architecture

```text
Trusted client application
    │
    │ 1. Gets access to the Column Master Key
    │ 2. Driver decrypts the Column Encryption Key
    │ 3. Driver encrypts parameters / decrypts results
    ↓
SQL Database Engine
    │ stores ciphertext
    │ stores key metadata and encrypted CEK values
    │ does not receive ordinary plaintext keys
    ↓
Data files and backups contain encrypted column values
```

### Key types

| Key | Purpose | Storage |
|---|---|---|
| Column Master Key (CMK) | Key-protecting key that protects one or more CEKs | Trusted store outside the database, such as Azure Key Vault, Windows certificate store, or HSM. |
| Column Encryption Key (CEK) | Encrypts values in encrypted columns | Stored in database metadata only in encrypted form. |

### Client requirements

- Use an Always Encrypted-capable driver.
- Enable Always Encrypted in the connection, commonly with `Column Encryption Setting=enabled`.
- Give the application access to the CMK in the external key store.
- Parameterize values sent to encrypted columns.
- Do not expect plaintext literals or ordinary T-SQL variables to work in comparisons with encrypted columns.

### Deterministic versus randomized encryption

| Property | Deterministic | Randomized |
|---|---|---|
| Same plaintext produces same ciphertext | Yes | No |
| Information leakage | More pattern/frequency leakage | Stronger protection against pattern analysis |
| Equality point lookup | Supported | Not without a secure enclave |
| Equality join/group/index | Supported within documented restrictions | Not without a secure enclave |
| Range and pattern search | Not supported without enclave | Supported for enclave-enabled randomized columns, subject to version restrictions |
| Preferred when search is unnecessary | No | Yes |

Choose randomized encryption by default when equality search is not required. Use deterministic encryption only when its supported equality operations are necessary and the leakage tradeoff is acceptable.

### Encrypted column definition pattern

```sql
CREATE TABLE dbo.Patient
(
    PatientId INT PRIMARY KEY,
    NationalId CHAR(11) COLLATE Latin1_General_BIN2
        ENCRYPTED WITH
        (
            COLUMN_ENCRYPTION_KEY = CEK_Patient,
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL
);
```

The key objects and encrypted key material must already be provisioned correctly. Tools such as SSMS, PowerShell, and `SqlPackage` can automate key provisioning and data encryption.

### Common Always Encrypted traps

- The Database Engine does not provision plaintext Always Encrypted keys.
- Parameterization is required for values targeting encrypted columns.
- The driver must have Always Encrypted enabled.
- Key-store access and database permissions are separate requirements.
- Deterministic encryption enables equality operations but exposes value patterns.
- Randomized encryption without an enclave does not support searching, grouping, indexing, or joining.
- Unsupported data types and operations must be checked during design.
- Key rotation requires a planned operational process.

## 8. Always Encrypted with secure enclaves

A **secure enclave** is a protected region of memory inside the SQL environment. It allows the Database Engine to perform approved operations on plaintext and keys without exposing them to the rest of the engine, operating system, or administrators.

### Why enclaves exist

Ordinary Always Encrypted intentionally gives the engine very limited capability because it cannot see plaintext. Secure enclaves add two major capabilities:

1. **In-place cryptographic operations**
2. **Rich confidential queries**

### In-place cryptographic operations

An enclave can:

- Encrypt an existing plaintext column in place.
- Re-encrypt a column with a new enclave-enabled CEK.
- Rotate a CEK.
- Change the encryption type, such as deterministic to randomized.
- Decrypt an enclave-enabled column in place.

Without an enclave, these operations normally require moving data to a trusted client for decryption and re-encryption.

### Rich confidential queries

Depending on the platform and version, enclave-enabled randomized columns can support:

- Comparison operators.
- `BETWEEN` and `IN`.
- `LIKE` pattern matching.
- `DISTINCT`.
- Joins.
- `ORDER BY` and `GROUP BY` in supported SQL Server 2022/Azure SQL configurations.
- Nonclustered indexes on enclave-enabled randomized encrypted columns.

> [!IMPORTANT]
> Rich enclave computations use **randomized encryption**. Deterministic encryption continues to use equality operations outside the enclave model. Check platform, SQL Server version, compatibility level, collation, driver, and enclave configuration.

### Enclave technologies

| Platform | Documented enclave technology |
|---|---|
| SQL Server 2019 and later on Windows | Virtualization-based Security (VBS) enclave |
| Azure SQL Database | VBS or Intel SGX depending on service configuration and hardware |

Secure enclaves are not listed as a general Azure SQL Managed Instance capability in the referenced secure-enclave documentation.

### Attestation

Attestation is a defense-in-depth mechanism that lets a client verify that:

- it is communicating with a genuine supported enclave; and
- the expected Microsoft-signed enclave code is running.

The client establishes trust before releasing sensitive keys for enclave computation. Attestation requirements and supported services vary by platform and enclave type.

### Ordinary Always Encrypted versus secure enclaves

| Capability | Always Encrypted | With secure enclave |
|---|---|---|
| Plaintext exposed to ordinary engine memory | No | No |
| Plaintext temporarily available inside protected enclave | No | Yes |
| Equality on deterministic columns | Yes | Yes |
| `LIKE`, ranges, richer comparisons on randomized columns | No | Yes, when supported |
| In-place encryption and key rotation | No | Yes |
| Index randomized encrypted column | No | Yes, when enclave-enabled and supported |

## 9. Dynamic Data Masking

Dynamic Data Masking (DDM) changes query results for users who lack `UNMASK`. It does **not** change or encrypt the stored data.

### Masking functions

| Function | Purpose | Example |
|---|---|---|
| `default()` | Full mask based on data type | String becomes `XXXX`; numeric becomes `0`. |
| `email()` | Email-shaped mask | `aXXX@XXXX.com` |
| `partial(prefix,"padding",suffix)` | Reveal selected prefix/suffix | `partial(0,"XXX-XX-",4)` |
| `random(start,end)` | Random value in a range for numeric columns | `random(1,100)` |
| `datetime("unit")` | Mask a selected date/time component | `datetime("Y")`; SQL Server 2022+ |

### Create masked columns

```sql
CREATE TABLE dbo.CustomerContact
(
    CustomerId INT PRIMARY KEY,
    FullName   NVARCHAR(100) NOT NULL,
    Email      NVARCHAR(200)
        MASKED WITH (FUNCTION = 'email()') NOT NULL,
    Phone      VARCHAR(30)
        MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)') NULL,
    CreditLimit DECIMAL(12,2)
        MASKED WITH (FUNCTION = 'random(100,500)') NULL,
    BirthDate DATE
        MASKED WITH (FUNCTION = 'datetime("Y")') NULL
);
```

Add or remove a mask:

```sql
ALTER TABLE dbo.CustomerContact
ALTER COLUMN FullName ADD MASKED
    WITH (FUNCTION = 'partial(1,"XXXX",1)');

ALTER TABLE dbo.CustomerContact
ALTER COLUMN FullName DROP MASKED;
```

### `SELECT` and `UNMASK` are separate

```sql
CREATE ROLE SupportReader;
GRANT SELECT ON OBJECT::dbo.CustomerContact TO SupportReader;

-- The role can select rows but sees masked values.

GRANT UNMASK ON dbo.CustomerContact(Email) TO SupportReader;
-- The role can now see the actual Email, but other masked columns stay masked.
```

Starting with SQL Server 2022, `UNMASK` can be granted at database, schema, table, or column scope.

### DDM limitations and security meaning

- DDM is not encryption.
- Stored values remain unchanged.
- Users with `CONTROL`, `db_owner`, or `sysadmin` can see unmasked values.
- A user also needs `SELECT`; `UNMASK` alone does not grant data access.
- A mask does not prevent updates if the user has `UPDATE` permission.
- Determined users with broad query access may infer values through exhaustive predicates.
- DDM cannot be applied directly to Always Encrypted columns.
- Use DDM with permissions, RLS, auditing, and encryption.

Discover masked columns:

```sql
SELECT
    OBJECT_SCHEMA_NAME(object_id) AS SchemaName,
    OBJECT_NAME(object_id) AS TableName,
    name AS ColumnName,
    masking_function
FROM sys.masked_columns
WHERE is_masked = 1;
```

## 10. Row-Level Security

Row-Level Security (RLS) restricts which rows are visible or writable based on user identity, group membership, or session context.

### Two predicate types

| Predicate | Effect |
|---|---|
| Filter predicate | Silently removes unauthorized rows from `SELECT`, `UPDATE`, and `DELETE` visibility. |
| Block predicate | Rejects disallowed `INSERT`, `UPDATE`, or `DELETE` operations with an error. |

RLS uses:

1. A schema-bound inline table-valued predicate function.
2. A security policy that attaches the function to a table.

### Tenant isolation example

```sql
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_TenantAccess(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS IsAllowed
    WHERE @TenantId = TRY_CAST(SESSION_CONTEXT(N'TenantId') AS INT)
);
GO

CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantAccess(TenantId)
    ON dbo.CustomerOrders,
ADD BLOCK PREDICATE Security.fn_TenantAccess(TenantId)
    ON dbo.CustomerOrders AFTER INSERT,
ADD BLOCK PREDICATE Security.fn_TenantAccess(TenantId)
    ON dbo.CustomerOrders AFTER UPDATE
WITH (STATE = ON, SCHEMABINDING = ON);
GO
```

Set the session context after securely determining the tenant:

```sql
EXEC sys.sp_set_session_context
    @key = N'TenantId',
    @value = 42,
    @read_only = 1;
```

> [!WARNING]
> Do not trust a tenant identifier supplied directly by an untrusted client. The application must map the authenticated identity to an authorized tenant and prevent later changes. Connection pooling must also set or clear session context correctly.

### RLS behavior and traps

- The database applies the policy regardless of which application issues the query.
- Even `dbo`, `db_owner`, and the table owner are filtered while an enabled policy applies.
- A disabled policy provides no filtering or blocking.
- The predicate function should be simple and performant.
- Index the columns used by the predicate, such as `TenantId`.
- Test `SELECT`, `INSERT`, `UPDATE`, and `DELETE`; a filter predicate alone may not prevent every unauthorized write transition.
- Temporal history tables need their own security predicates.
- CDC gating roles and Change Tracking permissions can reveal information outside normal RLS query results.
- Microsoft Fabric and Azure Synapse Analytics currently support filter predicates but not block predicates.

### RLS versus DDM

| Question | RLS | DDM |
|---|---|---|
| Does the row appear? | Controls row visibility | Does not hide rows |
| Is the value changed in storage? | No | No |
| Unauthorized result | Row is absent | Row appears with masked value |
| Main object | Security policy + predicate function | Masking function on a column |

## 11. Object-level permissions and least privilege

### Principal and securable hierarchy

```text
Principals
├─ Server: login, server role
└─ Database: user, database role, application role

Securables
├─ Server
├─ Database
├─ Schema
└─ Object: table, view, procedure, function, external model
```

### `GRANT`, `DENY`, and `REVOKE`

| Statement | Meaning |
|---|---|
| `GRANT` | Adds a permission. |
| `DENY` | Explicitly prohibits a permission and normally overrides grants inherited from other memberships. |
| `REVOKE` | Removes a previous `GRANT` or `DENY`; it does not necessarily remove access obtained elsewhere. |

Common exam trap: use `REVOKE` to remove a grant. Do not use `DENY` unless an explicit prohibition is intended.

### Grant permissions through roles

```sql
CREATE ROLE OrderReaders;
GO

GRANT SELECT ON SCHEMA::Reporting TO OrderReaders;
GRANT EXECUTE ON OBJECT::dbo.GetOrderSummary TO OrderReaders;
DENY DELETE ON OBJECT::dbo.CustomerOrders TO OrderReaders;
GO

ALTER ROLE OrderReaders ADD MEMBER [app-reader-group];
```

Best practices:

- Grant permissions to roles, not many individual users.
- Use Microsoft Entra groups for human access.
- Use separate workload identities for applications and environments.
- Grant permissions at the narrowest practical scope.
- Group objects with similar access requirements into schemas.
- Grant `EXECUTE` on stored procedures instead of direct table access when the procedure is the intended boundary.
- Avoid broad roles such as `db_owner` and `db_datawriter` when narrower permissions work.
- Review ownership chaining and dynamic SQL behavior.

### Useful catalog views and functions

```sql
SELECT * FROM sys.database_principals;
SELECT * FROM sys.database_role_members;
SELECT * FROM sys.database_permissions;

SELECT HAS_PERMS_BY_NAME(
    'dbo.CustomerOrders',
    'OBJECT',
    'SELECT'
) AS CanSelect;
```

SQL Server also exposes server-level equivalents such as `sys.server_principals`, `sys.server_role_members`, and `sys.server_permissions`. Azure SQL Database and SQL database in Fabric have different server-level capabilities.

## 12. Passwordless database access

Microsoft Entra authentication supports identities for users, groups, applications, service principals, and managed identities.

### Why passwordless is preferred

- No database password is stored in source code or configuration.
- No application secret needs manual rotation.
- Token lifetime is limited.
- Access can be governed centrally with Microsoft Entra ID.
- Human identities can use MFA and Conditional Access.
- Group membership simplifies onboarding and offboarding.

### Managed identity types

| Type | Characteristics |
|---|---|
| System-assigned | Lifecycle is tied to one Azure resource. It is deleted with that resource. |
| User-assigned | Independent Azure resource that can be assigned to multiple workloads. |

### End-to-end passwordless authorization

```text
1. Enable managed identity on the application resource.
2. Create or map the Microsoft Entra identity in the database.
3. Add the database user to a least-privilege database role.
4. Use a token-capable SQL driver.
5. Connect with managed identity authentication.
```

Database setup pattern:

```sql
CREATE USER [orders-api-identity] FROM EXTERNAL PROVIDER;
GO

CREATE ROLE OrdersApiRole;
GRANT SELECT, INSERT, UPDATE ON OBJECT::dbo.CustomerOrders TO OrdersApiRole;
GRANT EXECUTE ON OBJECT::dbo.GetOrderSummary TO OrdersApiRole;
ALTER ROLE OrdersApiRole ADD MEMBER [orders-api-identity];
GO
```

Use `DefaultAzureCredential` in applications when appropriate. In development it can use a signed-in developer identity; in Azure it can discover the workload's managed identity without code changes.

> [!IMPORTANT]
> A managed identity does not automatically receive database or Azure resource permissions. It removes secret management; it does not remove authorization design.

## 13. Network security

For Azure SQL Database:

- The logical server denies connections until network access is allowed.
- Public access can be restricted by server- or database-level firewall rules.
- Virtual network rules restrict traffic to selected networks.
- Private Link gives the logical server a private endpoint in a virtual network.
- “Allow Azure services and resources to access this server” is broad; it can allow traffic from resources outside your subscription. Prefer narrower rules or private access where possible.
- Restrict outbound connectivity for external REST/model calls when supported.

Network isolation complements identity and database permissions; it does not replace them.

## 14. Auditing

Auditing records selected database and server events for investigation, monitoring, and compliance.

### SQL Server Audit components

```text
SQL Server Audit
    ├─ destination: audit file / Windows Application log / Security log
    ├─ Server Audit Specification
    │      └─ server-level action groups
    └─ Database Audit Specification
           └─ database actions or action groups
```

An audit and its specification are created disabled and must be enabled.

### SQL Server example

```sql
USE master;
GO

CREATE SERVER AUDIT SecurityAudit
TO FILE
(
    FILEPATH = 'D:\SqlAudit\',
    MAXSIZE = 1 GB,
    MAX_ROLLOVER_FILES = 20
)
WITH
(
    ON_FAILURE = CONTINUE
);
GO

ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);
GO

USE ApplicationDb;
GO

CREATE DATABASE AUDIT SPECIFICATION SensitiveDataAudit
FOR SERVER AUDIT SecurityAudit
ADD (SELECT ON OBJECT::dbo.CustomerContact BY public),
ADD (UPDATE ON OBJECT::dbo.CustomerContact BY public)
WITH (STATE = ON);
GO
```

Read file-target audit data:

```sql
SELECT *
FROM sys.fn_get_audit_file(
    'D:\SqlAudit\*.sqlaudit',
    DEFAULT,
    DEFAULT
);
```

### Azure SQL auditing destinations

Azure SQL Database can send audit events to:

- Azure Storage;
- Log Analytics; and
- Event Hubs.

Use Storage for long-term retention, Log Analytics for querying and alerts, and Event Hubs for streaming to a SIEM or other consumers. Multiple destinations can be configured.

### Audit design checklist

- Define which actions and principals are in scope.
- Minimize noise while meeting compliance and investigation needs.
- Protect audit destinations from modification and deletion.
- Separate audit administrators from audit readers.
- Set retention and archival policies.
- Monitor audit health and diagnostic-setting deletion.
- Use managed identity for Azure Storage audit destinations where supported.
- Parameterize sensitive values; statement text in an audit can itself contain sensitive data.
- Create alerts for high-risk events and repeated failures.
- Test that expected events are actually captured.

> [!NOTE]
> Azure SQL auditing is optimized for service availability and performance. It supports compliance and investigation but does not by itself guarantee compliance or prevent every attack.

## 15. Secure AI model endpoints

SQL Server 2025-era platforms can define external models and call HTTPS endpoints. This creates a data-exfiltration boundary and must be tightly controlled.

### Prefer managed identity

```sql
CREATE DATABASE SCOPED CREDENTIAL
    [https://<azure-openai-resource>.cognitiveservices.azure.com/]
WITH
    IDENTITY = 'Managed Identity',
    SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
GO
```

The managed identity must also receive the required Azure role on the target Azure AI resource. Database configuration alone is not enough.

External model pattern:

```sql
CREATE EXTERNAL MODEL SecureEmbeddingModel
WITH
(
    LOCATION = 'https://<azure-openai-resource>.cognitiveservices.azure.com/openai/deployments/<deployment>/embeddings?api-version=<version>',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = '<deployment-model-name>',
    CREDENTIAL = [https://<azure-openai-resource>.cognitiveservices.azure.com/]
);
GO

GRANT EXECUTE
ON EXTERNAL MODEL::SecureEmbeddingModel
TO EmbeddingJobRole;
GO
```

Important permission separation:

- `CREATE EXTERNAL MODEL` or `ALTER ANY EXTERNAL MODEL` controls model administration.
- `EXECUTE` on the external model controls who can use that model.
- `EXECUTE ANY EXTERNAL ENDPOINT` controls direct use of `sp_invoke_external_rest_endpoint`.

### Secure external calls

- Prefer managed identity over API keys.
- Store unavoidable secrets in secure credential stores, not source control.
- Grant the target Azure role at the smallest practical scope.
- Restrict who can create credentials, models, or call external endpoints.
- Allow only required HTTPS endpoint domains and paths.
- Restrict outbound network access when supported.
- Minimize the columns and rows sent to a model.
- Remove or tokenize PII and secrets before constructing prompts.
- Treat model responses as untrusted input.
- Audit model and external REST usage.
- Apply timeouts, retry limits, cost controls, and rate limits.
- Review data residency, retention, and provider terms.

### Direct REST invocation risk

`sp_invoke_external_rest_endpoint` can send database data to an external entity. Its use should be treated as a privileged exfiltration capability.

```sql
GRANT EXECUTE ANY EXTERNAL ENDPOINT TO ExternalModelCallerRole;
```

Grant this permission only to narrowly scoped principals. Prefer a controlled stored procedure that validates the payload and endpoint rather than exposing unrestricted direct use.

## 16. Secure REST and GraphQL endpoints

Data API builder (DAB) can expose database entities through REST and GraphQL.

### Three security layers

1. **Authentication** — validate who is calling, commonly through Microsoft Entra JWT tokens.
2. **API authorization** — define which roles can perform which actions on which entities and fields.
3. **Database authorization** — give the DAB database identity only the permissions it actually needs.

### Secure-by-default authorization

DAB entities have no permissions until access is configured. Avoid granting anonymous access unless it is an explicit requirement.

Conceptual configuration:

```json
{
  "entities": {
    "CustomerOrder": {
      "source": "dbo.CustomerOrders",
      "permissions": [
        {
          "role": "order-reader",
          "actions": [
            {
              "action": "read",
              "fields": {
                "include": ["OrderId", "OrderDate", "Status"]
              }
            }
          ]
        }
      ]
    }
  }
}
```

For named roles, the caller's token must contain the role claim, and the request selects the role with the documented `X-MS-API-ROLE` behavior.

### Endpoint checklist

- Use HTTPS only.
- Validate token issuer, audience, signature, expiry, and intended tenant.
- Use entity/action permissions for create, read, update, delete, or execute.
- Restrict fields containing sensitive information.
- Apply database policies or RLS for row filtering.
- Use pagination and bounded query options to limit resource exhaustion.
- Disable entities and stored procedures that should not be exposed.
- Keep database connection secrets outside configuration; prefer managed identity.
- Do not expose broad stored procedures that accept arbitrary SQL.
- Log requests without recording tokens or sensitive request bodies.
- Apply gateway protections such as throttling, quotas, WAF controls, and IP restrictions where appropriate.

### Fabric API for GraphQL

Fabric GraphQL security commonly requires authorization at two levels:

1. The caller needs permission to execute the GraphQL API item.
2. The effective identity or saved connection needs appropriate permissions on the underlying data source.

With SSO, the caller's identity is passed through and data-source permissions apply to that caller. With saved credentials, queries execute using the stored connection identity, so protect that identity and keep its permissions narrow.

## 17. Secure MCP endpoints

MCP lets AI agents discover and invoke tools. A database MCP server can turn natural-language intent into data operations, so its tool surface is a security boundary.

### Two directions of authentication

```text
MCP client or AI agent
    │
    │ inbound authentication
    │ client → MCP server
    ↓
SQL MCP server
    │
    │ outbound authentication
    │ MCP server → database
    ↓
SQL database
```

Configure both:

- **Inbound:** Use Microsoft Entra/OAuth JWT validation or a trusted gateway. Validate issuer and audience.
- **Outbound:** Prefer managed identity and grant the database identity only required entity/procedure permissions.

### MCP security checklist

- Expose only required tools and entities.
- Use read-only tools when writes are unnecessary.
- Require explicit user approval for consequential or write operations.
- Display and review generated SQL when the MCP server executes arbitrary T-SQL.
- Keep DDL and unrestricted SQL execution disabled unless explicitly required.
- Apply DAB role, entity, action, field, and policy controls.
- Use an item-scoped Fabric endpoint when the agent should be bound to one item.
- Treat tool descriptions, database text, and model output as untrusted data that can contain prompt injection.
- Never let retrieved content silently expand permissions or choose credentials.
- Log tool calls and correlate them to the authenticated identity.
- Apply rate limits, payload limits, timeouts, and result-size limits.
- Separate development and production identities and endpoints.

Fabric's hosted Data Warehouse MCP endpoint uses the signed-in identity and respects Fabric and SQL permissions; it does not bypass Fabric security. That still requires least privilege and careful review of generated write statements.

## 18. AI-assisted development security

When Copilot or another agent can read schemas, generate SQL, or invoke tools:

- Do not paste credentials, personal data, or production records into prompts.
- Review generated SQL before execution.
- Check for destructive statements, missing predicates, unsafe dynamic SQL, and privilege escalation.
- Treat instructions found in files, database rows, and web content as untrusted.
- Use allowlisted tools with narrow scopes.
- Separate read-only exploration from write-capable automation.
- Require approval for schema changes and DML against production.
- Use branch protection, pull requests, code owners, and secret scanning.
- Audit both human and agent actions.
- Remember that an AI assistant's access is determined by the identity and tools you give it.

## 19. Scenario-selection table

| Scenario | Best answer |
|---|---|
| Stolen database files must remain unreadable | TDE |
| Network traffic must be encrypted | TLS |
| DBAs must not see national ID values | Always Encrypted |
| Equality search on encrypted column without enclave | Deterministic Always Encrypted |
| Stronger encrypted-column protection and no search needed | Randomized Always Encrypted |
| Range/`LIKE` queries on randomized encrypted data | Always Encrypted with secure enclave |
| Rotate an Always Encrypted key without moving all data to client | Secure enclave in-place cryptographic operation |
| Help-desk user may see rows but not full email/phone | Dynamic Data Masking |
| Each tenant may see only its own rows | Row-Level Security |
| User must be completely unable to select a column | Column/object permissions or a restricted view—not DDM alone |
| Application must connect without a stored password | Managed identity + Microsoft Entra database user |
| Record sensitive-table queries for investigation | SQL auditing |
| Application calls Azure OpenAI without an API key | Managed identity + database-scoped credential + Azure RBAC |
| REST role may read only selected fields | DAB role/action/field permissions |
| Agent connects to database MCP endpoint | Secure inbound auth and least-privilege outbound database identity |

## 20. Common exam traps

1. **Calling DDM encryption.** It only masks result values.
2. **Expecting TDE to hide data from authorized DBAs.** TDE protects physical storage, not normal query access.
3. **Using TDE without backing up the certificate.** Restores can become impossible.
4. **Confusing column-level encryption with Always Encrypted.** Column-level T-SQL encryption occurs in the engine; Always Encrypted is client-side.
5. **Storing a Column Master Key in the database.** The CMK lives in a trusted external key store; database metadata points to it.
6. **Choosing randomized Always Encrypted for equality search without an enclave.** That search is not supported.
7. **Choosing deterministic encryption without acknowledging pattern leakage.** Repeated plaintext produces repeated ciphertext.
8. **Assuming an enclave exposes plaintext to administrators.** Plaintext is processed inside protected enclave memory.
9. **Granting `UNMASK` but not `SELECT`.** `UNMASK` does not grant row access.
10. **Using only an RLS filter predicate to secure writes.** Add appropriate block predicates where the platform supports them.
11. **Assuming table owners bypass RLS.** Enabled policies still apply.
12. **Using `DENY` to remove a grant.** Use `REVOKE`; `DENY` is an explicit prohibition.
13. **Giving an application `db_owner`.** Grant narrow permissions through a custom role.
14. **Assuming managed identity grants access automatically.** Create/map the identity and assign database and Azure permissions.
15. **Storing API keys in source code.** Prefer managed identity; otherwise use a secure secret store.
16. **Securing an API only at the HTTP layer.** The database identity also needs least privilege, and row/field rules may be required.
17. **Securing only one side of MCP authentication.** Protect client-to-server and server-to-database paths.
18. **Trusting AI-generated SQL automatically.** Review and approve consequential operations.
19. **Treating auditing as prevention.** Auditing records activity; it does not itself block it.
20. **Logging sensitive tokens or prompts.** Audit and observability systems also need data-minimization controls.

## 21. Quick-reference cheat sheet

```text
Data in transit                  TLS
Files/logs/backups at rest       TDE
Selected values via T-SQL        Column/cell-level encryption
Hide plaintext from DBA          Always Encrypted
Rich encrypted queries           Always Encrypted + secure enclave
Hide displayed values            Dynamic Data Masking
Hide unauthorized rows           Row-Level Security
Allow an action                  GRANT
Explicitly prohibit an action    DENY
Remove GRANT or DENY              REVOKE
Passwordless Azure workload      Managed identity
Record database activity         SQL auditing
Call model without API key       Managed identity credential
Secure REST/GraphQL               AuthN + role/action/field rules + least privilege
Secure MCP                        Inbound auth + outbound auth + constrained tools
```

### Five words to anchor the exam

```text
TDE       → files
DDM       → display
RLS       → rows
AE        → columns
Audit     → evidence
```

## 22. Practice questions

### Question 1

A company must protect database files and backups if storage media is stolen, without changing application queries. Which feature is the best fit?

A. Dynamic Data Masking  
B. Transparent Data Encryption  
C. Row-Level Security  
D. `DENY SELECT`

<details>
<summary>Answer</summary>

**B.** TDE transparently protects data and log files at rest and TDE-protected backups.

</details>

### Question 2

Database administrators must be unable to read customer national IDs in plaintext. Which feature is designed for this requirement?

A. TDE  
B. DDM  
C. Always Encrypted  
D. SQL Audit

<details>
<summary>Answer</summary>

**C.** Always Encrypted keeps plaintext and plaintext keys out of the ordinary Database Engine environment.

</details>

### Question 3

An encrypted column requires equality lookups, but secure enclaves aren't available. Which Always Encrypted mode supports the lookup?

A. Randomized  
B. Deterministic  
C. TDE  
D. Partial masking

<details>
<summary>Answer</summary>

**B.** Deterministic encryption supports equality operations, with the tradeoff of greater pattern leakage.

</details>

### Question 4

You need `LIKE` and range searches over a randomized encrypted column. Which feature should you evaluate?

A. Always Encrypted with secure enclaves  
B. TDE  
C. DDM `partial()`  
D. A database audit specification

<details>
<summary>Answer</summary>

**A.** Secure enclaves enable richer confidential queries over supported enclave-enabled randomized columns.

</details>

### Question 5

What is a major operational benefit of a secure enclave?

A. It removes the need for encryption keys  
B. It automatically grants database permissions  
C. It supports in-place encryption and key rotation  
D. It disables auditing

<details>
<summary>Answer</summary>

**C.** The enclave can perform supported cryptographic operations without moving the data to a client.

</details>

### Question 6

A support role may query all customer rows but should see only masked email addresses. Which feature is the direct fit?

A. RLS  
B. DDM with `email()`  
C. TDE  
D. Backup encryption

<details>
<summary>Answer</summary>

**B.** DDM changes the displayed value for users without `UNMASK`.

</details>

### Question 7

What does granting `UNMASK` without `SELECT` allow?

A. Full access to the table  
B. Unmasked access only through views  
C. No data access by itself  
D. Update access

<details>
<summary>Answer</summary>

**C.** `UNMASK` changes how already-accessible data is displayed; it does not grant `SELECT`.

</details>

### Question 8

Each customer must see only rows belonging to its tenant. Which feature is designed for this?

A. DDM  
B. RLS  
C. TDE  
D. `email()` masking

<details>
<summary>Answer</summary>

**B.** RLS applies a database security predicate to restrict rows.

</details>

### Question 9

Which RLS predicate silently removes unauthorized rows from result sets?

A. Block predicate  
B. Filter predicate  
C. Audit predicate  
D. Mask predicate

<details>
<summary>Answer</summary>

**B.** A filter predicate controls row visibility. Block predicates reject prohibited writes.

</details>

### Question 10

You want to prevent a tenant from inserting a row for another tenant. What should supplement the RLS filter predicate?

A. An appropriate `AFTER INSERT` block predicate  
B. An email mask  
C. A TDE certificate  
D. `REVOKE UNMASK`

<details>
<summary>Answer</summary>

**A.** The block predicate rejects writes whose resulting tenant value violates the security rule.

</details>

### Question 11

A role inherited `SELECT` through another role. You want to remove one earlier direct grant without explicitly prohibiting access from every source. Which statement should you use?

A. `DENY`  
B. `REVOKE`  
C. `DROP USER`  
D. `UNMASK`

<details>
<summary>Answer</summary>

**B.** `REVOKE` removes the earlier grant. `DENY` is an explicit prohibition that normally overrides other grants.

</details>

### Question 12

An Azure application needs to connect to Azure SQL without a stored password. Which design is preferred?

A. Embed a SQL password in the connection string  
B. Store the password in an environment variable only  
C. Managed identity plus a Microsoft Entra database user and least-privilege role  
D. Grant `db_owner` to `public`

<details>
<summary>Answer</summary>

**C.** Managed identity provides passwordless authentication, but the identity still needs database authorization.

</details>

### Question 13

What additional permission does a managed identity need when calling Azure OpenAI?

A. No Azure permission because the identity exists  
B. An appropriate Azure RBAC role on the Azure OpenAI resource  
C. `UNMASK` on every database  
D. `CONTROL SERVER` in SQL

<details>
<summary>Answer</summary>

**B.** Authentication and authorization are separate. The identity needs the relevant Azure role at the target resource.

</details>

### Question 14

Which permission controls direct use of `sp_invoke_external_rest_endpoint`?

A. `EXECUTE ANY EXTERNAL ENDPOINT`  
B. `ALTER ANY MASK`  
C. `VIEW SERVER STATE`  
D. `UNMASK`

<details>
<summary>Answer</summary>

**A.** This permission is powerful because the procedure can transfer data to an external endpoint.

</details>

### Question 15

Which three destinations are available for Azure SQL auditing?

A. Azure Storage, Log Analytics, and Event Hubs  
B. Key Vault, DNS, and Private Link  
C. GitHub, npm, and Docker Hub  
D. Azure OpenAI, Search, and Maps

<details>
<summary>Answer</summary>

**A.** Choose the destination based on retention, analysis, alerting, and streaming requirements.

</details>

### Question 16

What must be secured for a Data API builder endpoint?

A. Only the HTTP endpoint  
B. Only the database login  
C. Authentication, API authorization, and the database identity's permissions  
D. Only the GraphQL schema name

<details>
<summary>Answer</summary>

**C.** Endpoint security is layered. A valid token should not imply unlimited entity or database access.

</details>

### Question 17

A Fabric GraphQL caller can authenticate but gets a permission error against a table. What should you check?

A. Only the TLS cipher  
B. API execute permission and underlying data-source permission  
C. The TDE certificate password  
D. Whether DDM uses `email()`

<details>
<summary>Answer</summary>

**B.** GraphQL API authorization and data-source authorization are separate layers.

</details>

### Question 18

Which two authentication paths must be considered for SQL MCP Server?

A. Browser to DNS and DNS to storage  
B. Client to MCP server and MCP server to database  
C. Database to TDE certificate and TDE certificate to backup  
D. Model to prompt and prompt to output

<details>
<summary>Answer</summary>

**B.** Secure inbound client authentication and outbound database authentication separately.

</details>

### Question 19

Why should AI-generated write SQL require explicit review or approval?

A. AI tools cannot generate SQL syntax  
B. Model output is nondeterministic and may contain destructive or overbroad operations  
C. TDE blocks AI queries  
D. `SELECT` always modifies data

<details>
<summary>Answer</summary>

**B.** Tool access can turn an incorrect or manipulated response into a consequential database operation.

</details>

### Question 20

Which statement best describes auditing?

A. It encrypts database files  
B. It masks columns  
C. It records selected activity for monitoring and investigation  
D. It replaces permissions

<details>
<summary>Answer</summary>

**C.** Auditing provides evidence and visibility; it does not replace preventative controls.

</details>

## 23. Official references

- [DP-800 official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [SQL Server encryption overview](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/sql-server-encryption?view=sql-server-ver17)
- [Transparent Data Encryption](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/transparent-data-encryption?view=sql-server-ver17)
- [Encrypt a column of data](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/encrypt-a-column-of-data?view=sql-server-ver17)
- [Always Encrypted](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine?view=sql-server-ver17)
- [Always Encrypted with secure enclaves](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves?view=sql-server-ver17)
- [Dynamic Data Masking](https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking?view=sql-server-ver17)
- [Row-Level Security](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security?view=sql-server-ver17)
- [Database Engine permissions](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/getting-started-with-database-engine-permissions?view=sql-server-ver17)
- [Microsoft Entra authentication for Azure SQL](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-overview?view=azuresql)
- [SQL Server Audit](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine?view=sql-server-ver17)
- [Azure SQL auditing](https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview?view=azuresql)
- [CREATE EXTERNAL MODEL](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql?view=sql-server-ver17)
- [`sp_invoke_external_rest_endpoint`](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql?view=sql-server-ver17)
- [Data API builder authorization](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authentication-local)
- [Fabric API for GraphQL application security](https://learn.microsoft.com/en-us/fabric/data-engineering/connect-apps-api-graphql)
- [SQL MCP Server overview](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview)
- [Fabric Data Warehouse MCP server](https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehouse-mcp-server)

---

## Final memory aid

> **Authenticate the identity, authorize the action, minimize the data, encrypt the right boundary, and audit the result.**
