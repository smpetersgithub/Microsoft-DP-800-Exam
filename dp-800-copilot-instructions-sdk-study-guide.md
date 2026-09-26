# DP-800 Study Guide: Copilot, Instruction Files, MCP, and SDK-Style SQL Projects

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topics:** GitHub Copilot, Copilot in Fabric, custom instructions, MCP tools, and `Microsoft.Build.Sql` SDK-style SQL Database Projects  
> **Last reviewed:** September 26, 2026

## Exam objectives covered

This guide addresses DP-800 objectives that require you to:

- Interpret the security impact of AI-assisted development tools.
- Enable GitHub Copilot and Copilot in Microsoft Fabric.
- Select chat, model, execution-mode, and Model Context Protocol (MCP) tool options.
- Create and configure GitHub Copilot instruction files.
- Connect to SQL Server and Fabric Lakehouse MCP endpoints.
- Create, build, and validate SQL Database Projects, including SDK-style projects.
- Produce, inspect, and deploy `.dacpac` artifacts through controlled CI/CD pipelines.
- Detect schema drift and protect deployment secrets.

> [!NOTE]
> In the DP-800 blueprint, **SDK-style** refers to SQL Database Projects based on the `Microsoft.Build.Sql` project SDK. It does not mean a general-purpose Copilot programming SDK.

## 1. The complete mental model

```text
Developer request
      ↓
GitHub Copilot or Copilot in Fabric
      ↓
Repository instructions + selected model + permitted tools/MCP context
      ↓
Proposed T-SQL or project change
      ↓
Human review + tests + SQL project build validation
      ↓
.dacpac deployment artifact
      ↓
Deploy report/script + approval
      ↓
SqlPackage publish to the target database
```

The components solve different problems:

| Component | Primary responsibility |
|---|---|
| Copilot | Generates explanations, plans, code, and proposed changes |
| Instruction file | Supplies persistent project conventions and workflow context |
| Model selection | Balances capability, latency, reliability, and usage cost |
| MCP/tool configuration | Controls which external capabilities and data sources an agent can call |
| SQL Database Project | Declarative, source-controlled database model |
| `Microsoft.Build.Sql` | Cross-platform SDK used to build and validate an SDK-style SQL project |
| `.dacpac` | Versioned deployment artifact containing the database model |
| SqlPackage | Compares, scripts, reports, and publishes database-model changes |
| CI/CD controls | Enforce tests, reviews, approvals, identity, and deployment policy |

> [!IMPORTANT]
> Copilot instructions influence model behavior; they do not enforce security. Authentication, authorization, branch protection, tool permissions, database permissions, and deployment approvals are the actual controls.

## 2. GitHub Copilot for Microsoft SQL development

### Main capabilities

With the MSSQL extension for Visual Studio Code, GitHub Copilot can help:

- explain T-SQL, views, procedures, functions, and schemas;
- generate queries, constraints, indexes, tests, and mock data;
- identify SQL injection and overly broad permission patterns;
- analyze execution plans and suggest possible query improvements;
- create or modify database objects;
- scaffold data-access code and object-relational mapping integrations;
- configure Data API builder REST, GraphQL, and MCP endpoints;
- work with SQL Server, Azure SQL, and Microsoft Fabric SQL endpoints.

Generated content can be incomplete, insecure, invalid, or inefficient. Always build, test, inspect the execution plan where relevant, and review changes before deployment.

### Basic enablement

For an organization-managed GitHub Copilot environment:

1. The enterprise or organization enables the required Copilot policies and models.
2. A Copilot license/seat is assigned to the user when required by the plan.
3. The user installs or enables GitHub Copilot and Copilot Chat in the supported IDE.
4. For SQL-aware workflows in Visual Studio Code, install the MSSQL extension.
5. Sign in to GitHub and authorize the extensions.
6. Connect the MSSQL extension to the correct server and database with a least-privilege identity.

Enterprise policy takes precedence over organization settings. Users cannot override a feature or model disabled by their administrator.

### Choose the correct interaction mode

| Mode | Best use | Expected behavior |
|---|---|---|
| Ask | Explain a query, answer a question, or generate a one-shot suggestion | Primarily conversational; no autonomous multi-file implementation |
| Edit | Apply a focused change across selected files | Proposes direct file edits within a controlled working set |
| Plan | Analyze a broad requirement before implementation | Produces a structured plan for review |
| Agent | Complete a multi-step task using files, terminal commands, database tools, or MCP | Selects tools and iterates; can make changes subject to host controls |
| Inline completion | Finish code while the user types | Fast ghost-text suggestion based mainly on editor context |

### Schema-aware versus non-schema-aware suggestions

| Surface | Knows the connected database schema? | Use case |
|---|:---:|---|
| Inline completion in a `.sql` file | No | Generic SQL patterns and boilerplate |
| `@mssql` chat participant | Yes | Queries using actual connected tables and columns |
| MSSQL agent-mode tools | Yes | Multi-step database exploration and actions |
| Schema Designer | Yes | AI-assisted schema design |
| Ordinary Copilot chat without database context | Not necessarily | Repository/code questions |

> [!TIP]
> If the exam asks for generated SQL that must use real table and column names from a connected database, prefer the schema-aware `@mssql` participant or MSSQL agent tools—not inline completion.

### Model selection

Available model names and policies change over time. Select by workload characteristics rather than memorizing a temporary model list.

| Requirement | Model-choice principle |
|---|---|
| Simple completion or explanation | Favor lower latency and lower usage cost |
| Complex schema redesign or debugging | Favor stronger reasoning and code capability |
| Long repository context | Choose a model with sufficient context support |
| Organization-controlled environment | Choose only an administrator-approved model |
| No clear preference | Use automatic model selection when allowed |

Higher reasoning settings can improve difficult tasks but usually increase latency and consumption. Model selection does not replace validation.

## 3. Copilot in Microsoft Fabric SQL database

### Copilot experiences

In the Fabric SQL query editor, Copilot provides:

- a **chat pane** for natural-language questions and generated SQL;
- **inline code completion** while writing T-SQL;
- **Explain** and **Fix** quick actions for selected SQL;
- schema-grounded responses based on the user's accessible database context.

### Enable Copilot in Fabric

Typical requirements include:

1. A paid, supported Fabric capacity—normally F2 or higher.
2. The Fabric tenant administrator enables the relevant Copilot/AI tenant settings.
3. If settings are delegated, the capacity administrator enables the corresponding capacity settings.
4. The workspace is assigned to a Copilot-enabled capacity.
5. Users receive appropriate workspace and item access.
6. Cross-region or subprocessor settings are reviewed and enabled only when permitted by organizational policy.

Enabling Copilot for a tenant does not grant users access to every Fabric item. Copilot operates with the permissions of the current user.

### Execution modes

| Fabric Copilot mode | Behavior |
|---|---|
| Read-only | Can run safe read queries; drafts DDL/DML changes without executing them |
| Read and write with approval | Can execute DDL or DML only after the user reviews and approves it |

`SELECT` queries can execute automatically. Statements that modify schema or data require approval in read/write mode.

### Context and data sent to Copilot

Fabric SQL Copilot can use:

- the accessible database schema;
- SQL executed by the user;
- related query error messages;
- previous messages and responses in the current session.

Microsoft states that prompts and responses from this experience aren't used to retrain the underlying Azure OpenAI foundation models. Administrators must still evaluate geographic-processing, retention, compliance, and tenant-policy requirements.

### GitHub Copilot versus Fabric Copilot

| Scenario | Better starting tool |
|---|---|
| Work across a repository, tests, app code, and a SQL project | GitHub Copilot in the IDE |
| Ask questions directly in the Fabric SQL query editor | Copilot in Fabric |
| Generate a multi-file pull request | GitHub Copilot agent workflow |
| Explain or fix a query in a Fabric database | Fabric Copilot quick action/chat |
| Validate a declarative database model | SDK-style SQL project build |

## 4. GitHub Copilot instruction files

### Purpose

Custom instructions provide standing repository context so every prompt does not have to repeat:

- architecture and project layout;
- target SQL platform;
- naming and coding conventions;
- security requirements;
- approved commands;
- build, test, and review steps;
- prohibited patterns and deployment safeguards.

Copilot might not follow an instruction identically every time because model output is nondeterministic. Treat the file as guidance, not policy enforcement.

### Instruction and prompt types

| Type | Location | Scope |
|---|---|---|
| Repository-wide Copilot instructions | `.github/copilot-instructions.md` | Copilot work throughout the repository |
| Path-specific instructions | `.github/instructions/<name>.instructions.md` | Files matching the `applyTo` glob |
| Agent instructions | `AGENTS.md` or supported equivalent | Standing rules for supported agents, often scoped by directory hierarchy |
| Prompt file | Usually `*.prompt.md` in a supported workspace location | Reusable prompt for a specific task, invoked when needed |
| Personal/organization instructions | Product or organization settings | User- or organization-level behavior where supported |

Support varies across GitHub.com, Visual Studio Code, Visual Studio, JetBrains, Copilot CLI, cloud agent, and code review. Verify the surface named in the question.

### Repository-wide example

Create `.github/copilot-instructions.md`:

```markdown
# Database project context

- The repository's source of truth is `src/Contoso.Database/Contoso.Database.sqlproj`.
- The target is Azure SQL Database.
- Build with `dotnet build src/Contoso.Database/Contoso.Database.sqlproj -c Release`.
- Treat build warnings as review items; never hide errors by lowering the target platform.

# T-SQL conventions

- Use schema-qualified, two-part object names.
- Use `NVARCHAR` for user-facing Unicode text.
- Use `DATETIME2` rather than legacy `DATETIME` for new objects.
- Parameterize external input. Never concatenate input into executable SQL.
- Avoid `SELECT *` in production views and stored procedures.
- Include `SET NOCOUNT ON` in stored procedures unless row-count messages are required.

# Safety and security

- Never place credentials, connection strings, access tokens, or production data in source files.
- Do not generate destructive deployment options without explicitly describing the risk.
- Prefer Microsoft Entra workload identity or managed identity for automation.
- Apply least-privilege database permissions.

# Validation

- Build the SQL project after changing database objects.
- Add or update tests for changed behavior.
- Generate a deployment script or deploy report before publishing to production.
- Summarize schema changes and possible data-loss risks in the pull request.
```

### Path-specific example

Create `.github/instructions/sql.instructions.md`:

```markdown
---
applyTo: "**/*.sql"
---

- Write idempotent pre-deployment and post-deployment logic where practical.
- Do not use `NOLOCK` as a default performance fix.
- Never use dynamic SQL for user input without `sp_executesql` parameters.
- Qualify objects with their schema.
- Consider transaction scope, locking, error handling, and rollback behavior.
- Explain any operation that can drop, truncate, or rewrite data.
```

### What belongs in an instruction file

- stable repository facts;
- commands that actually work in the repository;
- target platform and supported versions;
- required test and build workflow;
- schema and naming rules;
- security and data-handling expectations;
- links or paths to authoritative project documentation.

### What does not belong

- passwords, tokens, or connection strings;
- temporary task details that belong in the current prompt;
- enormous copies of general documentation;
- contradictory or impossible rules;
- vague statements such as “write perfect SQL”;
- assumptions that an instruction prevents unauthorized tool use.

### Instruction-file exam traps

1. Repository-wide instructions use `.github/copilot-instructions.md`.
2. Path-specific files must end in `.instructions.md` under `.github/instructions/` and use `applyTo` frontmatter.
3. A prompt file is invoked for a task; repository instructions are standing context.
4. Instructions improve consistency but do not guarantee compliance.
5. Enforcement belongs in linters, tests, build gates, permissions, and protected pipelines.

## 5. Model Context Protocol (MCP) and tool configuration

### Definition

MCP is an open protocol that lets an AI host discover and invoke external tools and access approved context.

```text
User prompt
    ↓
AI host/client (Copilot)
    ↓ discovers tools and submits structured arguments
MCP server
    ↓ authenticates and applies permissions
SQL database, OneLake, source control, or another service
```

### Main terms

| Term | Meaning |
|---|---|
| Host | The AI application containing the conversation, such as an IDE |
| MCP client | The host component that communicates with a server |
| MCP server | A service that exposes approved capabilities |
| Tool | A callable operation with structured input and output |
| Resource | Context the server can expose for reading, when supported by the client |
| Prompt | Reusable prompt template exposed by a server, when supported |
| Transport | Communication mechanism such as standard input/output or streamable HTTP |

### Tool-selection principles

- Enable only the servers and tools required for the task.
- Start with read-only tools.
- Disable create, update, delete, permission-management, and command-execution tools unless needed.
- Give the authenticated identity the least privilege possible.
- Review the host's approval behavior; different Copilot surfaces handle approval differently.
- Treat tool output as untrusted data that could contain misleading or prompt-injection content.
- Log and monitor tool calls where supported.
- Keep credentials outside repository configuration.

Fewer enabled tools improve security, reduce context usage, and can improve tool-selection accuracy.

### SQL MCP Server through Data API builder

Microsoft's SQL MCP Server is included with Data API builder. It exposes configured tables, views, or stored procedures as controlled data-manipulation tools.

The `dab-config.json` controls:

- the database connection through a secret reference or environment variable;
- which entities are exposed;
- which columns are available;
- role-based permissions;
- allowed create, read, update, and delete actions;
- entity and field descriptions that help the model choose valid operations.

SQL MCP Server focuses on DML over exposed entities. Use SQL project tooling or the MSSQL extension for schema/DDL development.

### Connect a local SQL MCP endpoint to Visual Studio Code

A simple HTTP `.vscode/mcp.json` entry is:

```json
{
  "servers": {
    "sql-mcp-server": {
      "type": "http",
      "url": "http://localhost:5000/mcp"
    }
  }
}
```

For local development, standard-input/output transport can let Visual Studio Code manage the DAB process:

```json
{
  "servers": {
    "sql-mcp-server": {
      "type": "stdio",
      "command": "dab",
      "args": [
        "start",
        "--mcp-stdio",
        "role:reader",
        "--loglevel",
        "error",
        "--config",
        "${workspaceFolder}/dab-config.json"
      ]
    }
  }
}
```

Do not store a database password in `mcp.json` or commit a secret-filled `.env` file. Use environment/secret references, appropriate ignore rules, and a least-privilege role.

### SQL MCP Server security example

| Requirement | Configuration decision |
|---|---|
| Agent only needs product lookup | Expose the product view with read permission only |
| Cost price is sensitive | Exclude that column from the exposed entity |
| Deletes are never allowed | Disable/remove delete tools and deny delete permission in SQL |
| Multi-tenant application | Enforce authorization and row filtering server-side; never rely on the prompt |
| Production access | Use strong authentication, managed secrets, TLS, logging, and network controls |

### Fabric Lakehouse/OneLake MCP

The Fabric MCP server's OneLake tools can:

- list accessible workspaces and Fabric items;
- browse and read files;
- discover lakehouse namespaces, tables, columns, and metadata;
- write, upload, download, or delete files when the identity is permitted;
- create some Fabric items.

The local OneLake tools use the signed-in Azure identity and existing Fabric permissions. They do not grant extra workspace access.

### Fabric Core MCP versus OneLake tools

| Server/tool family | Primary purpose |
|---|---|
| Fabric Core MCP remote endpoint | Manage Fabric workspaces, items, permissions, folders, and capacity information |
| OneLake/Fabric local MCP tools | Browse files and inspect table structures/data-plane content in lakehouses and other OneLake-backed items |
| SQL MCP Server | Perform controlled DML against configured SQL entities |

The Fabric Core remote MCP endpoint is for resource management, not general-purpose querying or editing of lakehouse table data.

### MCP security trap

An MCP server expands what an AI agent can do. It does not automatically make those actions safe. Depending on the host, configured MCP tools might be called autonomously or might require approval. Use server-side authorization as the final boundary.

## 6. Security impact of AI-assisted SQL development

### Main risks and mitigations

| Risk | Example | Mitigation |
|---|---|---|
| Hallucinated schema | Query references a nonexistent column | Use schema-aware context; build and test |
| Destructive SQL | Suggested `DROP`, `TRUNCATE`, or broad `DELETE` | Use read-only mode, approvals, backups, and script review |
| SQL injection | Generated string concatenation | Require parameterized queries and security tests |
| Secret exposure | Connection string pasted into chat or committed | Secret manager, environment references, content exclusion, secret scanning |
| Excessive permissions | Agent connects as owner/admin | Least-privilege identity and separate environments |
| Sensitive-data exposure | Production row data becomes model context | Use approved data, masking, test data, tenant policy, and governance |
| Prompt injection | Database text or tool output tells the model to misuse tools | Treat external content as data, restrict tools, require approvals |
| Supply-chain risk | Untrusted MCP server executes code | Use trusted servers, pin versions, inspect packages, sandbox |
| Incorrect optimization | New index helps one query but hurts writes | Test representative workload and execution plans |
| Compliance/region issue | Data processed outside an approved boundary | Review and configure Fabric/GitHub data policies before enabling |

### Safe workflow

```text
Use nonproduction data and least privilege
          ↓
Ask Copilot for a proposal/explanation
          ↓
Review generated T-SQL and tool calls
          ↓
Build and run automated tests
          ↓
Inspect deployment script/report
          ↓
Peer approval and controlled deployment
          ↓
Monitor and retain audit evidence
```

### Content exclusion

GitHub Copilot Business and Enterprise can exclude selected repository paths from supported Copilot features. Content exclusion is useful but not absolute:

- support differs among chat, edit, agent, review, and IDE surfaces;
- an IDE might indirectly provide semantic information derived from an excluded file;
- exclusions don't replace repository permissions or secret removal.

Never commit a secret merely because a path is excluded from Copilot.

## 7. SDK-style SQL Database Projects

### Definition

A SQL Database Project is a source-controlled, declarative representation of a database schema. The SDK-style format uses the `Microsoft.Build.Sql` project SDK.

Building the project:

1. parses the project's T-SQL files;
2. validates supported syntax for the selected target platform;
3. validates object relationships and references; and
4. produces a `.dacpac` model artifact.

### Why use SDK-style projects

- Cross-platform build through the .NET SDK.
- Recommended format for new SQL project development.
- Used by the SQL Database Projects extension for Visual Studio Code.
- Default globbing includes `.sql` files without listing each file manually.
- Supports NuGet package references for database dependencies.
- Fits GitHub Actions and other CI/CD systems.
- Separates declarative database state from environment-specific deployment.

### Minimal `.sqlproj`

```xml
<Project DefaultTargets="Build">
  <Sdk Name="Microsoft.Build.Sql" Version="<approved-version>" />

  <PropertyGroup>
    <Name>Contoso.Database</Name>
    <DSP>Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider</DSP>
  </PropertyGroup>
</Project>
```

Pin and test an approved SDK version rather than using an unreviewed floating dependency.

### Target platform (`DSP`)

The database schema provider controls which platform/version syntax the build accepts.

| Target | Example `DSP` value |
|---|---|
| SQL Server 2019 | `Microsoft.Data.Tools.Schema.Sql.Sql150DatabaseSchemaProvider` |
| SQL Server 2022 | `Microsoft.Data.Tools.Schema.Sql.Sql160DatabaseSchemaProvider` |
| SQL Server 2025 | `Microsoft.Data.Tools.Schema.Sql.Sql170DatabaseSchemaProvider` |
| Azure SQL Database | `Microsoft.Data.Tools.Schema.Sql.SqlAzureV12DatabaseSchemaProvider` |

Do not change the target merely to hide errors. It must match the intended deployment platform.

### Typical project structure

```text
Contoso.Database/
├── Contoso.Database.sqlproj
├── Security/
│   └── AppReader.sql
├── Tables/
│   ├── Customer.sql
│   └── Order.sql
├── Views/
│   └── CustomerSummary.sql
├── StoredProcedures/
│   └── GetCustomer.sql
└── Scripts/
    ├── PreDeployment.sql
    └── PostDeployment.sql
```

Folders are organizational. The declarative model and object dependencies—not file execution order—determine normal deployment ordering.

### Build and artifact

```powershell
dotnet build .\Contoso.Database.sqlproj -c Release
```

The normal result is similar to:

```text
bin/Release/Contoso.Database.dacpac
```

A `.dacpac` contains the database model and deployment metadata. It is not a normal full backup and does not automatically contain production table data.

### What a build catches

- invalid T-SQL for the selected target;
- references to missing tables, columns, or other modeled objects;
- some naming/casing and code-analysis warnings;
- invalid database references.

### What a build does not prove

- that a query returns correct business results;
- that a migration preserves production data;
- that performance is acceptable at production scale;
- that permissions meet the organization's policy;
- that generated code is safe from every injection or concurrency issue.

Use unit tests, integration tests, security analysis, deployment review, and workload testing in addition to the build.

### Database/package references

SDK-style projects can use NuGet package references for database models:

```xml
<ItemGroup>
  <PackageReference
      Include="Microsoft.SqlServer.Dacpacs.Master"
      Version="<approved-version>" />
</ItemGroup>
```

Package references are the recommended reference style for new SDK-style project development. Use appropriate metadata when the referenced model represents another database or another server.

### Declarative deployment

SQL projects describe the **desired final schema**. SqlPackage compares the `.dacpac` model with the target and creates an incremental deployment plan.

| SqlPackage action | Purpose |
|---|---|
| `Publish` | Apply changes so the target matches the `.dacpac` model |
| `Script` | Generate the incremental deployment T-SQL without applying it |
| `DeployReport` | Generate an XML report of changes that publish would make |
| `DriftReport` | Report changes made to a registered database since its last registration |
| `Extract` | Create a `.dacpac` or SQL project from an existing database |

### Review before deployment

```powershell
SqlPackage /Action:Script `
  /SourceFile:".\bin\Release\Contoso.Database.dacpac" `
  /TargetConnectionString:"<target-connection-string>" `
  /DeployScriptPath:".\artifacts\deployment.sql"
```

```powershell
SqlPackage /Action:DeployReport `
  /SourceFile:".\bin\Release\Contoso.Database.dacpac" `
  /TargetConnectionString:"<target-connection-string>" `
  /DeployReportPath:".\artifacts\deploy-report.xml"
```

Do not place literal production credentials in scripts or workflow YAML. Supply authentication securely at runtime.

### Publish

```powershell
SqlPackage /Action:Publish `
  /SourceFile:".\bin\Release\Contoso.Database.dacpac" `
  /TargetConnectionString:"<secure-runtime-value>"
```

For sensitive environments, publish only after the generated plan/script, automated checks, and required approvals succeed.

### Schema drift

**Schema drift** occurs when someone changes a deployed database outside the approved project/pipeline, causing it to differ from the controlled model or last registered deployment.

```powershell
SqlPackage /Action:DriftReport `
  /TargetConnectionString:"<secure-runtime-value>" `
  /OutputPath:".\artifacts\drift-report.xml"
```

Drift detection should normally alert or block a release for review. Automatically overwriting unexplained drift can destroy emergency fixes or unmodeled objects.

## 8. CI/CD workflow for an SDK-style SQL project

```text
Feature branch
      ↓
Copilot-assisted change following repository instructions
      ↓
Pull request + code owners + peer review
      ↓
dotnet build + tests + static/security checks
      ↓
Versioned .dacpac pipeline artifact
      ↓
Drift check + deployment script/report
      ↓
Environment approval
      ↓
SqlPackage Publish with workload identity
      ↓
Verification, monitoring, and rollback/recovery procedure
```

### Recommended controls

- Require pull requests and protected branches.
- Assign database specialists through `CODEOWNERS` for schema paths.
- Build once and promote the same `.dacpac` through environments.
- Separate build identity from deployment identity.
- Prefer OpenID Connect, managed identity, or another short-lived/passwordless mechanism.
- Store nonsecret environment differences in publish profiles or pipeline variables.
- Store secrets in the platform's protected secret store.
- Generate and review scripts/reports for production.
- Require approval for high-risk environments.
- Detect schema drift before deployment.
- Back up or confirm recovery capability before destructive changes.
- Record the artifact version and deployment result.

### Reference/static data

Schema projects primarily model schema. Carefully controlled reference data can be deployed through reviewed post-deployment scripts, often using idempotent `MERGE`-style or explicit upsert logic. Do not treat large transactional datasets as project source files.

### Pre- and post-deployment scripts

- **Pre-deployment:** runs before the generated deployment plan; use sparingly for prerequisites.
- **Post-deployment:** runs after the model change; useful for reference data or carefully controlled transitions.
- Included scripts aren't independently compiled into the declarative object model.
- Scripts must be repeatable or safely guarded when pipelines can rerun.

## 9. Putting Copilot, instructions, MCP, and the SDK together

### Example scenario

Requirement: Add a customer-preference table and a read-only MCP surface for customer-support agents.

1. Repository instructions tell Copilot the Azure SQL target, naming rules, secure data types, and build command.
2. Use Plan mode to outline the schema, permissions, tests, and DAB exposure.
3. Use Agent/Edit mode to create the table file and tests in a feature branch.
4. Copilot proposes DAB configuration exposing only approved columns with read permission.
5. A developer reviews privacy and row-filtering requirements.
6. `dotnet build` validates the SQL project and produces a `.dacpac`.
7. Automated tests confirm constraints and expected queries.
8. The pipeline creates a deployment script or deploy report.
9. Code owners approve the pull request and production deployment.
10. SqlPackage publishes using a least-privilege workload identity.

### Responsibility map

| Concern | Correct mechanism |
|---|---|
| Suggest table design | Copilot |
| Communicate naming and safety conventions | Instruction files |
| Read actual SQL schema or data through tools | MSSQL context or approved MCP tools |
| Prevent unauthorized data access | Database/MCP authentication and authorization |
| Validate declarative object relationships | SQL project build |
| Package database model | `.dacpac` |
| Preview deployment changes | SqlPackage `Script` or `DeployReport` |
| Detect out-of-band schema changes | SqlPackage `DriftReport` or schema comparison |
| Enforce review and approval | GitHub branch/pipeline controls |

## 10. Common exam traps

1. **Copilot output is a proposal.** Human review, tests, and build validation remain necessary.
2. **Inline SQL completion is not schema-aware.** Use `@mssql` or MSSQL agent tools for connected schema context.
3. **Instructions are not access control.** MCP and database permissions enforce access.
4. **Repository instructions and prompt files have different scopes.** Standing rules versus reusable task prompts.
5. **Path-specific instructions require `applyTo`.** Store them under `.github/instructions/` with an `.instructions.md` suffix.
6. **Agent mode can use tools and modify state.** Ask mode is a safer fit for explanations or one-shot suggestions.
7. **Fabric read-only mode drafts DDL/DML but does not execute it.** Read/write mode still requires approval.
8. **Copilot uses the current user's accessible context.** Enabling Copilot doesn't grant new database or Fabric permissions.
9. **MCP increases capability and risk.** Enable only required tools with least privilege.
10. **Tool approval behavior differs by host.** Never assume every MCP call prompts the user.
11. **A SQL project is declarative.** Normal object-file order isn't deployment order.
12. **`dotnet build` creates a `.dacpac`.** It does not deploy the database.
13. **`Publish` changes the target.** `Script` and `DeployReport` are review-oriented and non-publishing.
14. **Drift is an out-of-band database change.** Investigate it before overwriting the target.
15. **A `.dacpac` is not a regular database backup.** It represents a model, not complete operational data.
16. **Target platform matters.** Build validation uses the selected `DSP`.
17. **Build success doesn't prove runtime correctness or performance.** Tests and workload evaluation are still required.
18. **Secrets do not belong in instructions, prompts, source files, or committed MCP configuration.**

## 11. Memory aids

### Copilot workflow

```text
Ask → Review → Build → Test → Report → Approve → Publish
```

### Instruction files

```text
Repository-wide → .github/copilot-instructions.md
Path-specific   → .github/instructions/*.instructions.md + applyTo
Agent-wide      → AGENTS.md where supported
Reusable task   → *.prompt.md
```

### SqlPackage actions

```text
Build        → creates .dacpac
Script       → shows deployment T-SQL
DeployReport → describes intended deployment changes
DriftReport  → finds out-of-band changes
Publish      → changes the target database
Extract      → creates a model from an existing database
```

### Security rule

```text
Instructions guide.
Permissions enforce.
Approvals control change.
Tests provide evidence.
```

## 12. Practice questions

### Question 1

Which file supplies repository-wide GitHub Copilot instructions?

A. `.github/copilot-instructions.md`  
B. `.github/workflows/copilot.yml`  
C. `.vscode/settings.sql`  
D. `database.dacpac`

<details>
<summary>Answer</summary>

**A. `.github/copilot-instructions.md`.**

</details>

### Question 2

Which frontmatter property scopes a path-specific instruction file?

A. `targetPlatform`  
B. `applyTo`  
C. `databaseName`  
D. `publish`

<details>
<summary>Answer</summary>

**B. `applyTo`.** It contains glob patterns for the files to which the instructions apply.

</details>

### Question 3

A developer needs an explanation of a stored procedure without autonomous file or database changes. Which mode is the best starting choice?

A. Agent mode  
B. Ask mode  
C. Publish mode  
D. Drift mode

<details>
<summary>Answer</summary>

**B. Ask mode.** It is optimized for questions and explanations.

</details>

### Question 4

Which GitHub Copilot SQL surface uses the schema of a connected database?

A. Generic inline ghost-text completion  
B. `@mssql` chat participant  
C. A plain Markdown preview  
D. Git history

<details>
<summary>Answer</summary>

**B. `@mssql`.** Inline completion does not see the connected database schema.

</details>

### Question 5

In Fabric Copilot's read-only mode, what happens when a user asks it to create a table?

A. The table is created without review.  
B. Copilot drafts SQL but doesn't execute the DDL.  
C. The request enables write mode.  
D. The database is exported.

<details>
<summary>Answer</summary>

**B.** Read-only mode can propose the statement but doesn't execute schema- or data-changing SQL.

</details>

### Question 6

What is the correct security boundary for an MCP tool that queries customer data?

A. A sentence in the prompt  
B. Model choice  
C. Server-side authentication, authorization, and database permissions  
D. The color theme of the IDE

<details>
<summary>Answer</summary>

**C.** Instructions and prompts aren't enforceable authorization boundaries.

</details>

### Question 7

An agent only needs to look up product descriptions. What is the safest initial MCP configuration?

A. Enable every tool with database-owner credentials.  
B. Expose a read-only product entity using a least-privilege role.  
C. Permit arbitrary shell commands.  
D. Place the administrator password in `mcp.json`.

<details>
<summary>Answer</summary>

**B.** Minimize the tool surface and permissions.

</details>

### Question 8

Which Fabric MCP capability is intended primarily for lakehouse file and table discovery?

A. OneLake MCP tools  
B. SqlPackage `Publish`  
C. GitHub branch protection  
D. CDC cleanup job

<details>
<summary>Answer</summary>

**A. OneLake MCP tools.** They can browse accessible OneLake items, files, and table metadata.

</details>

### Question 9

What is the project SDK recommended for new SDK-style SQL Database Projects?

A. `Microsoft.Build.Sql`  
B. `Microsoft.AspNetCore.App`  
C. `Microsoft.NET.Test.Sdk` only  
D. `Azure.Storage.Blobs`

<details>
<summary>Answer</summary>

**A. `Microsoft.Build.Sql`.**

</details>

### Question 10

What does `dotnet build` normally produce from an SDK-style SQL project?

A. A production backup  
B. A `.dacpac`  
C. A live database connection  
D. An audit log

<details>
<summary>Answer</summary>

**B. A `.dacpac`.** It contains the compiled database model and deployment metadata.

</details>

### Question 11

Which project property identifies the SQL target platform?

A. `DSP`  
B. `MCP`  
C. `LSN`  
D. `RRF`

<details>
<summary>Answer</summary>

**A. `DSP`.** The database schema provider determines platform-specific build validation.

</details>

### Question 12

Which SqlPackage action generates deployment T-SQL without applying it?

A. `Publish`  
B. `Script`  
C. `Extract`  
D. `DriftReport`

<details>
<summary>Answer</summary>

**B. `Script`.** Use it to review the proposed incremental deployment SQL.

</details>

### Question 13

Which action reports changes made to a registered database outside its last registered deployment?

A. `DriftReport`  
B. `Publish`  
C. `Build`  
D. `VectorSearch`

<details>
<summary>Answer</summary>

**A. `DriftReport`.**

</details>

### Question 14

A SQL project builds successfully. What has **not** been proven?

A. The T-SQL parses for the target model.  
B. Modeled references can be resolved.  
C. The queries meet production performance and business requirements.  
D. A `.dacpac` can be created.

<details>
<summary>Answer</summary>

**C.** Build validation does not replace runtime, data, security, and performance tests.

</details>

### Question 15

What is the safest way to supply a production deployment credential to a pipeline?

A. Store it in `.github/copilot-instructions.md`.  
B. Commit it in a publish script.  
C. Use a protected secret or short-lived workload identity at runtime.  
D. Include it in the pull-request description.

<details>
<summary>Answer</summary>

**C.** Prefer passwordless/short-lived identity when possible and otherwise use a protected secret store.

</details>

### Question 16

Why should a pipeline build once and promote the same `.dacpac` through environments?

A. To ensure each environment receives a separately generated schema  
B. To improve consistency and artifact traceability  
C. To avoid source control  
D. To bypass deployment approval

<details>
<summary>Answer</summary>

**B.** Promoting the same reviewed artifact reduces environmental variation and provides traceability.

</details>

### Question 17

Which statement about custom instructions is correct?

A. They guarantee that Copilot obeys every rule.  
B. They replace database permissions.  
C. They provide persistent project guidance but require independent enforcement.  
D. They should contain production passwords.

<details>
<summary>Answer</summary>

**C.** Model behavior is nondeterministic; enforce requirements through code, tests, policies, and permissions.

</details>

### Question 18

An organization needs to prevent unreviewed production schema changes. Which combination provides the strongest control?

A. A prompt asking Copilot to be careful  
B. Repository instructions only  
C. Protected branches, code-owner review, build/tests, deployment report, environment approval, and least-privilege deployment identity  
D. A faster AI model

<details>
<summary>Answer</summary>

**C.** These controls enforce review, validation, identity, and release policy independently of model behavior.

</details>

## 13. Rapid-review checklist

- [ ] I can choose Ask, Edit, Plan, or Agent mode from a scenario.
- [ ] I know inline SQL completion isn't connected-schema-aware.
- [ ] I know how `@mssql` and MSSQL tools obtain schema context.
- [ ] I can explain Fabric Copilot read-only and read/write-with-approval modes.
- [ ] I understand the tenant, capacity, workspace, and user-access layers for Fabric Copilot.
- [ ] I know the location of repository-wide and path-specific instruction files.
- [ ] I can write correct `applyTo` frontmatter.
- [ ] I can distinguish instructions, `AGENTS.md`, and reusable prompt files.
- [ ] I know instructions guide but don't enforce behavior.
- [ ] I can describe MCP hosts, clients, servers, and tools.
- [ ] I can choose a least-privilege MCP tool configuration.
- [ ] I can distinguish SQL MCP Server, OneLake tools, and Fabric Core MCP.
- [ ] I understand prompt-injection and secret-exposure risks.
- [ ] I know `Microsoft.Build.Sql` is the SDK for SDK-style SQL projects.
- [ ] I know the purpose of the `DSP` target-platform property.
- [ ] I know that `dotnet build` validates the model and creates a `.dacpac`.
- [ ] I can distinguish `Script`, `DeployReport`, `DriftReport`, `Publish`, and `Extract`.
- [ ] I can explain build-once/promote-many CI/CD.
- [ ] I can identify controls for secrets, approvals, and schema drift.

## 14. Official references

### DP-800 and AI-assisted SQL

- [DP-800 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [Implement SQL solutions by using AI-assisted tools](https://learn.microsoft.com/en-us/training/modules/design-implement-sql-solutions-ai-assisted-tools/)
- [GitHub Copilot for the MSSQL extension](https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/github-copilot/overview?view=sql-server-ver17)
- [Inline GitHub Copilot completions in SQL files](https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/github-copilot/inline-completions?view=sql-server-ver17)

### Copilot instructions and security

- [About customizing GitHub Copilot responses](https://docs.github.com/en/copilot/concepts/prompting/response-customization)
- [Add repository custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions)
- [Custom-instruction support by surface](https://docs.github.com/en/copilot/reference/custom-instructions-support)
- [GitHub Copilot enterprise and organization policies](https://docs.github.com/en/copilot/concepts/enterprise/policies)
- [Content exclusion for GitHub Copilot](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/content-exclusion)

### Copilot in Fabric

- [Enable and configure Copilot in Fabric](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-enable-fabric)
- [Copilot in Fabric SQL database FAQ](https://learn.microsoft.com/en-us/fabric/database/sql/copilot-faq)
- [Privacy and security for Fabric SQL Copilot](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-database-privacy-security)

### MCP

- [About MCP in GitHub Copilot](https://docs.github.com/en/copilot/concepts/context/mcp)
- [SQL MCP Server overview](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview)
- [Data API builder in the MSSQL extension](https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-data-api-builder?view=sql-server-ver17)
- [Use AI agents with OneLake through MCP](https://learn.microsoft.com/en-us/fabric/onelake/onelake-local-mcp)
- [Fabric Core MCP Server overview](https://learn.microsoft.com/en-us/rest/api/fabric/articles/mcp-servers/core-remote/overview-core-mcp-server)

### SDK-style SQL projects

- [What are SQL Database Projects?](https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-database-projects?view=sql-server-ver17)
- [SQL project target platforms](https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/concepts/target-platform?view=sql-server-ver17)
- [SQL project automation](https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-projects-automation?view=sql-server-ver17)
- [SqlPackage overview and actions](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage?view=sql-server-ver17)
- [Deploy and drift reports](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-deploy-drift-report?view=sql-server-ver17)

---

> **One-sentence exam summary:** Use Copilot to assist—not authorize—SQL development, encode stable project conventions in the correct instruction files, expose only least-privilege MCP tools, and rely on `Microsoft.Build.Sql`, `.dacpac` artifacts, testing, review reports, protected pipelines, and SqlPackage for controlled database delivery.
