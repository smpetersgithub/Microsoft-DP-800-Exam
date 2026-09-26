# DP-800 Study Guide: Vector Databases and Vector Search

> **Exam:** DP-800 — Developing AI-Enabled Database Solutions  
> **Topic:** Intelligent search with vectors in Microsoft SQL platforms  
> **Last reviewed:** September 26, 2026

## Exam objectives covered

This guide addresses the DP-800 skills related to intelligent search:

- Choose among full-text, semantic vector, and hybrid search.
- Design vector storage, including the `VECTOR` data type, dimensions, precision, and size.
- Use `VECTOR_DISTANCE`, `VECTOR_SEARCH`, `VECTOR_NORMALIZE`, `VECTOR_NORM`, and `VECTORPROPERTY`.
- Choose between exact nearest-neighbor search and approximate nearest-neighbor search.
- Select a vector index and distance metric.
- Understand how embeddings, chunking, vector retrieval, and retrieval-augmented generation work together.

> [!IMPORTANT]
> Approximate vector indexes and some AI-related T-SQL features are version- and platform-dependent, and some remain in preview. For an exam question, pay close attention to the named platform, feature status, workload size, accuracy requirement, and latency requirement.

## 1. Core idea

A **vector** is an ordered array of numbers. An **embedding** is a vector created by an AI model to represent meaningful features of data such as text, images, or audio.

Semantically similar inputs normally produce embeddings that are close together in vector space. A vector search compares a query embedding with stored embeddings and returns the closest matches.

```text
Source text
   ↓
Embedding model
   ↓
[0.018, -0.142, 0.731, ...]
   ↓
Store vector beside the source data
   ↓
Compare it with a query vector
   ↓
Return the nearest records
```

### Essential terms

| Term | Meaning |
|---|---|
| Vector | An ordered numeric array. |
| Embedding | A vector whose values encode learned features or meaning. |
| Dimension | The number of elements in a vector. |
| Embedding model | The model that converts source data into embeddings. |
| Vector store | A system that stores vectors and supports similarity search. |
| Similarity search | Retrieval based on closeness in vector space rather than exact keyword equality. |
| Query vector | The embedding generated from the user's query. |
| Nearest neighbors | Stored vectors closest to the query vector according to a distance metric. |
| `k` | The number of nearest results requested. |
| Recall | The proportion of true nearest neighbors found by an approximate search. |

### Vector database versus vector-capable SQL database

A separate, specialized vector database is not always required. SQL Server 2025, Azure SQL Database, eligible Azure SQL Managed Instance configurations, and SQL database in Microsoft Fabric can store vectors beside relational data and search them with T-SQL.

This is useful when an application needs both:

- vector similarity; and
- joins, transactions, security, filters, constraints, and ordinary SQL queries.

## 2. Embeddings and dimensions

An embedding model determines the output dimension. If a model produces 1,536 values, the target column and the query variable must be compatible with that dimension.

```sql
CREATE TABLE dbo.DocumentChunks
(
    ChunkId      BIGINT IDENTITY PRIMARY KEY,
    DocumentId   BIGINT NOT NULL,
    ChunkText    NVARCHAR(MAX) NOT NULL,
    Category     NVARCHAR(100) NULL,
    Embedding    VECTOR(1536) NOT NULL
);
```

### Rules to remember

1. Stored and query vectors must have compatible dimensions.
2. Use the **same embedding model and configuration** for the indexed content and the query.
3. If the embedding model or output dimension changes, regenerate the affected embeddings.
4. Update an embedding when the source content represented by it changes.
5. More dimensions increase storage, memory, I/O, and computation; they do not automatically guarantee a better application.
6. Store useful filter metadata, such as tenant, language, category, and security scope, beside the vector.

> [!WARNING]
> Comparing embeddings produced by unrelated models usually has no useful meaning, even when their dimension counts happen to match.

## 3. The SQL `VECTOR` data type

The native syntax is:

```sql
VECTOR(<dimensions>)
```

The default base type is `float32`. SQL stores vectors in an optimized binary representation and exposes them in a JSON-array form for convenience.

```sql
DECLARE @v1 VECTOR(3) = '[1.0, -0.2, 30.0]';
DECLARE @v2 VECTOR(3) = JSON_ARRAY(1.0, -0.2, 30.0);

SELECT @v1 AS VectorOne,
       @v2 AS VectorTwo;
```

### Storage sizing

For `float32`, each dimension uses 4 bytes. Microsoft documentation gives an additional 8-byte vector header, so a useful estimate is:

```text
Estimated vector bytes = (dimensions × 4) + 8
```

Examples:

| Dimensions | Approximate vector storage |
|---:|---:|
| 384 | 1,544 bytes |
| 768 | 3,080 bytes |
| 1,024 | 4,104 bytes |
| 1,536 | 6,152 bytes |

The native type supports from 1 through 1,998 dimensions. SQL Server 2025 also documents `float16` support as a preview feature; it reduces storage and precision. Treat its availability as platform/version-specific.

### Inspect vector properties

```sql
DECLARE @v VECTOR(3) = '[1, 2, 3]';

SELECT VECTORPROPERTY(@v, 'Dimensions') AS DimensionCount,
       VECTORPROPERTY(@v, 'BaseType')   AS BaseType;
```

`VECTORPROPERTY` is an exam-friendly choice when the question asks for the dimension count or base type of a vector.

## 4. Distance metrics

A distance metric converts the relationship between two vectors into a numeric value. For SQL vector distance results, **smaller means closer**.

| Metric | T-SQL name | What it emphasizes | Typical use |
|---|---|---|---|
| Cosine distance | `'cosine'` | Difference in direction; magnitude is largely ignored | Semantic text similarity; common default for embeddings |
| Euclidean distance | `'euclidean'` | Straight-line geometric distance, including magnitude | Spatial or feature vectors where absolute position/scale matters |
| Negative dot product | `'dot'` | Alignment and magnitude | Models or recommendation workloads designed for dot-product scoring |

### Important ranges

- Cosine distance: `0` means identical direction; `2` means opposing direction.
- Euclidean distance: `0` means identical vectors; the upper range is unbounded.
- SQL's `'dot'` metric returns a **negative dot product**, so smaller values indicate greater similarity.

```sql
DECLARE @a VECTOR(3) = '[1, 0, 0]';
DECLARE @b VECTOR(3) = '[0.8, 0.2, 0]';

SELECT VECTOR_DISTANCE('cosine',    @a, @b) AS CosineDistance,
       VECTOR_DISTANCE('euclidean', @a, @b) AS EuclideanDistance,
       VECTOR_DISTANCE('dot',       @a, @b) AS NegativeDotProduct;
```

### Exam decision rule

- If the question emphasizes **semantic meaning regardless of vector magnitude**, prefer cosine.
- If absolute geometric distance matters, prefer Euclidean.
- If the model was trained or documented for dot-product retrieval, use dot.
- The metric in an approximate search should match the metric used to create the vector index.

## 5. Exact versus approximate search

Microsoft documentation may use **exact nearest neighbor (ENN)** or **exact k-nearest neighbor (kNN)** for exhaustive search. The important distinction is whether every eligible vector is evaluated.

| Characteristic | Exact search | Approximate search |
|---|---|---|
| Main T-SQL feature | `VECTOR_DISTANCE` | `VECTOR_SEARCH` |
| Exam terminology | ENN or exact kNN | ANN |
| Search behavior | Calculates distance for every eligible vector | Uses a vector index to navigate to likely neighbors |
| Accuracy | Guaranteed exact nearest results | May miss some true nearest results |
| Performance | More CPU and latency as the candidate set grows | Faster and more scalable for large sets |
| Vector index used | No | Yes |
| Best fit | Small/filtered candidate set or maximum accuracy | Large candidate set with low-latency requirements |

### Exact search with `VECTOR_DISTANCE`

```sql
DECLARE @QueryVector VECTOR(1536) = @EmbeddingFromApplication;

SELECT TOP (10)
       ChunkId,
       DocumentId,
       ChunkText,
       VECTOR_DISTANCE('cosine', @QueryVector, Embedding) AS Distance
FROM dbo.DocumentChunks
WHERE Category = N'Database'
ORDER BY Distance;
```

`VECTOR_DISTANCE` always performs an exact calculation and does **not** use a vector index, even if one exists.

Microsoft gives fewer than 50,000 eligible vectors as a general point at which exact search can be appropriate. This is guidance, not a universal hard limit. Selectivity matters: a large table can still have a small exact-search candidate set after filters are applied.

### Approximate search with `VECTOR_SEARCH`

SQL Database Engine vector indexes use the **DiskANN** algorithm. DiskANN uses a graph-based structure designed for low-latency approximate nearest-neighbor retrieval while making efficient use of SSD storage and memory.

```sql
CREATE VECTOR INDEX IX_DocumentChunks_Embedding
ON dbo.DocumentChunks(Embedding)
WITH
(
    METRIC = 'cosine',
    TYPE = 'diskann'
);
```

Current syntax for a latest-version vector index:

```sql
DECLARE @QueryVector VECTOR(1536) = @EmbeddingFromApplication;

SELECT TOP (10) WITH APPROXIMATE
       d.ChunkId,
       d.DocumentId,
       d.ChunkText,
       s.Distance
FROM VECTOR_SEARCH
(
    TABLE      = dbo.DocumentChunks AS d,
    COLUMN     = Embedding,
    SIMILAR_TO = @QueryVector,
    METRIC     = 'cosine'
) AS s
WHERE d.Category = N'Database'
ORDER BY s.Distance;
```

Earlier vector-index versions use a `TOP_N` argument inside `VECTOR_SEARCH`. That syntax is deprecated for latest-version indexes:

```sql
-- Earlier-version syntax; know how to recognize it.
FROM VECTOR_SEARCH
(
    TABLE      = dbo.DocumentChunks AS d,
    COLUMN     = Embedding,
    SIMILAR_TO = @QueryVector,
    METRIC     = 'cosine',
    TOP_N      = 10
) AS s
```

> [!TIP]
> If the question says **guaranteed nearest records**, choose exact search. If it says **millions of vectors**, **low latency**, and a small recall tradeoff is acceptable, choose ANN with a vector index.

## 6. Recall, latency, and resource tradeoffs

**Recall** compares approximate results with the true results from an exact search:

```text
Recall = relevant exact neighbors found by ANN ÷ relevant exact neighbors
```

A recall of `1.0` means the approximate search returned all of the true nearest neighbors for the evaluated set.

Typical relationship:

```text
More exhaustive search effort
        ↑
Higher recall ───────── Higher latency/resource use
Lower recall  ───────── Lower latency/resource use
        ↓
Less exhaustive search effort
```

Evaluate a vector-search design with representative production data and queries. Measure at least:

- recall or relevance;
- query latency;
- throughput;
- CPU, memory, and I/O;
- index build and maintenance cost;
- embedding generation latency and cost.

## 7. Vector functions to know

| Function | Purpose | Key exam fact |
|---|---|---|
| `VECTOR_DISTANCE` | Calculates exact distance between two vectors | Exact; does not use a vector index |
| `VECTOR_SEARCH` | Returns approximate nearest neighbors | Uses an ANN/vector-index strategy |
| `VECTOR_NORM` | Returns vector magnitude | Supports `norm1`, `norm2`, and `norminf` |
| `VECTOR_NORMALIZE` | Rescales a vector to unit length | Keeps direction but changes magnitude to 1 |
| `VECTORPROPERTY` | Returns vector metadata | `Dimensions` and `BaseType` |

### Norm and normalization

```sql
DECLARE @v VECTOR(3) = '[1, 2, 3]';

SELECT VECTOR_NORM(@v, 'norm1')   AS L1Norm,
       VECTOR_NORM(@v, 'norm2')   AS L2Norm,
       VECTOR_NORM(@v, 'norminf') AS InfinityNorm,
       VECTOR_NORMALIZE(@v, 'norm2') AS UnitVector;
```

- `norm1`: sum of absolute component values.
- `norm2`: Euclidean magnitude.
- `norminf`: largest absolute component value.
- Azure OpenAI embedding models return normalized vectors, but other model sources might not. Check the model rather than normalizing blindly.

## 8. Generating embeddings

Embeddings can be generated:

- in application code through a model SDK or REST endpoint; or
- in supported Microsoft SQL platforms by using a database-defined external model and `AI_GENERATE_EMBEDDINGS`.

Conceptual T-SQL example:

```sql
UPDATE d
SET Embedding = AI_GENERATE_EMBEDDINGS(
                    d.ChunkText USE MODEL MyEmbeddingModel
                )
FROM dbo.DocumentChunks AS d
WHERE d.Embedding IS NULL;
```

The database model object holds the model endpoint definition. Access still requires appropriate credentials, permissions, networking, and endpoint configuration.

### Operational rules

- Batch background embedding generation for large ingestion jobs when possible.
- Avoid regenerating unchanged content.
- Record the model name/version and embedding status.
- Retry transient endpoint failures safely.
- Do not mix embeddings from incompatible models in the same search space.
- Re-embed data after a meaningful model migration.

## 9. Chunking and vector schema design

Large documents are normally divided into smaller chunks before embedding. Retrieval works at the chunk level so the system can return the most relevant passages instead of entire documents.

```text
Document → chunks → one embedding per chunk → retrieve top chunks
```

### Chunking tradeoffs

| Choice | Benefit | Risk |
|---|---|---|
| Smaller chunks | Precise retrieval | Context can be fragmented; more vectors and rows |
| Larger chunks | More context per result | Irrelevant material can dilute the embedding |
| Overlap | Preserves ideas across boundaries | More storage and duplicate retrieval |
| Metadata filters | Reduce irrelevant candidates and enforce scope | Poorly selective or post-search filtering can reduce usefulness |

### Recommended relational design

```sql
CREATE TABLE dbo.Documents
(
    DocumentId   BIGINT IDENTITY PRIMARY KEY,
    Title        NVARCHAR(300) NOT NULL,
    SourceUri    NVARCHAR(2048) NULL,
    ModifiedAt   DATETIME2 NOT NULL
);

CREATE TABLE dbo.DocumentChunks
(
    ChunkId          BIGINT IDENTITY PRIMARY KEY,
    DocumentId       BIGINT NOT NULL,
    ChunkOrdinal     INT NOT NULL,
    ChunkText        NVARCHAR(MAX) NOT NULL,
    TenantId         INT NOT NULL,
    LanguageCode     VARCHAR(10) NULL,
    EmbeddingModel   NVARCHAR(100) NOT NULL,
    EmbeddedAt       DATETIME2 NOT NULL,
    Embedding        VECTOR(1536) NOT NULL,

    CONSTRAINT FK_DocumentChunks_Documents
        FOREIGN KEY (DocumentId)
        REFERENCES dbo.Documents(DocumentId),

    CONSTRAINT UQ_DocumentChunks_Document_Ordinal
        UNIQUE (DocumentId, ChunkOrdinal)
);
```

This separates document metadata from chunk-level text and embeddings while preserving normal relational integrity.

## 10. Search method selection

| Requirement | Best starting choice | Reason |
|---|---|---|
| Exact words, product codes, names, or phrases | Full-text search | Lexical matching is precise for known terms |
| Same meaning with different wording | Vector search | Embeddings capture semantic similarity |
| Both exact terms and semantic meaning matter | Hybrid search | Combines lexical precision and semantic recall |
| Small candidate set and exact results required | Exact vector search | Exhaustive comparison guarantees nearest neighbors |
| Very large candidate set and low latency required | ANN vector search | Vector index trades a little recall for speed |

### Hybrid search

Hybrid search runs full-text and vector retrieval and combines their ranked results. A common fusion method is **Reciprocal Rank Fusion (RRF)**.

RRF uses result positions rather than trying to directly compare unlike scores such as a full-text BM25 score and a vector distance:

```text
RRF score(document) = Σ 1 / (constant + rank in each result list)
```

Use hybrid search when queries can contain both:

- exact identifiers or specialized terms; and
- natural-language descriptions or concepts.

Example: `DP-800 VECTOR_SEARCH low latency` benefits from exact matching on `DP-800` and `VECTOR_SEARCH` plus semantic matching for the performance intent.

## 11. Retrieval-augmented generation (RAG)

A vector store often provides the retrieval stage of a RAG application.

```text
INGESTION
Documents → chunk → embed → store text + vectors + metadata

QUERY
User question → embed → retrieve nearest chunks → add chunks to prompt → LLM response
```

### What each component does

| Component | Responsibility |
|---|---|
| Embedding model | Converts content and the query into comparable vectors |
| SQL vector store | Stores vectors, text, and metadata; returns nearest chunks |
| Filters/security | Restrict which chunks the requester is allowed to retrieve |
| Language model | Generates an answer from the question and retrieved context |

> [!IMPORTANT]
> Vector search retrieves relevant data; it does not itself generate the final natural-language answer.

## 12. Security and governance

- Treat embeddings as data derived from the source. Protect them according to the source data's sensitivity.
- Apply tenant and authorization filters during retrieval.
- Use least-privilege permissions for tables, model objects, credentials, and external endpoints.
- Keep secrets in supported credential mechanisms, not T-SQL source code.
- Log model/version lineage so embeddings can be reproduced or migrated.
- Establish a process to delete or regenerate embeddings when source records are deleted or changed.
- Validate external-model network access and data-governance requirements before sending content to a model endpoint.
- Do not assume semantic similarity is an authorization boundary.

## 13. Common exam traps

1. **A vector is not automatically an embedding.** An embedding is a vector produced to represent learned features or meaning.
2. **Dimension count is fixed by the model/configuration.** A `VECTOR(1536)` column cannot accept an incompatible output dimension.
3. **Same dimension does not imply compatibility.** Two different models can produce unrelated vector spaces of the same size.
4. **Lower distance means closer.** SQL's dot metric is the negative dot product for distance-style ordering.
5. **`VECTOR_DISTANCE` is exact.** It never uses the vector index.
6. **`VECTOR_SEARCH` is approximate.** It is the vector-index path and can trade recall for performance.
7. **Full-text and vector search solve different problems.** Full-text is lexical; vector search is semantic.
8. **Hybrid search is often strongest when both exact terms and meaning matter.**
9. **Filters can change the effective candidate count.** A selective filter can make exact search practical on a large table.
10. **Embeddings become stale.** Update them when their source content changes.
11. **Changing models is a data migration.** Existing content and queries must occupy the same vector space.
12. **Preview/version clues matter.** Choose syntax and features supported by the platform named in the question.

## 14. Memory aids

### Function mnemonic

```text
DISTANCE  = exact comparison
SEARCH    = approximate/indexed retrieval
NORM      = measure length
NORMALIZE = make length equal to 1
PROPERTY  = inspect metadata
```

### Search-choice mnemonic

```text
Words only     → full-text
Meaning only   → vector
Words + meaning → hybrid
```

### Exact/approximate mnemonic

```text
Exact = Every eligible vector
ANN   = Almost nearest, much faster at scale
```

## 15. Practice questions

### Question 1

An application must find support articles that mean the same thing as a user's issue even when none of the same words are used. Which search type is the best starting point?

A. Equality search  
B. Full-text search only  
C. Semantic vector search  
D. B-tree lookup

<details>
<summary>Answer</summary>

**C. Semantic vector search.** Embeddings enable retrieval based on conceptual similarity rather than requiring matching words.

</details>

### Question 2

A query must return the mathematically closest records from a filtered set of 8,000 embeddings. Approximation is not acceptable. Which function should you use?

A. `VECTOR_SEARCH`  
B. `VECTOR_DISTANCE`  
C. `VECTORPROPERTY`  
D. `VECTOR_NORMALIZE`

<details>
<summary>Answer</summary>

**B. `VECTOR_DISTANCE`.** Order exact distance calculations and select the top results. `VECTOR_SEARCH` performs approximate retrieval.

</details>

### Question 3

A table contains millions of embeddings, and the application requires low-latency retrieval. A small loss in recall is acceptable. What should you use?

A. An exact scan with `VECTOR_DISTANCE`  
B. `LIKE '%term%'`  
C. A DiskANN vector index with `VECTOR_SEARCH`  
D. A primary key lookup

<details>
<summary>Answer</summary>

**C.** Approximate nearest-neighbor search with a DiskANN vector index is designed for scalable, low-latency vector retrieval.

</details>

### Question 4

An embedding model returns 1,536 values. Which column definition is appropriate?

A. `VECTOR(1536)`  
B. `VECTOR(768)`  
C. `FLOAT(1536)`  
D. `NVARCHAR(1536)`

<details>
<summary>Answer</summary>

**A. `VECTOR(1536)`.** The vector column's dimension must match the embedding output.

</details>

### Question 5

Which function returns the number of dimensions in a vector?

A. `VECTOR_NORM`  
B. `VECTORPROPERTY`  
C. `VECTOR_DISTANCE`  
D. `JSON_VALUE`

<details>
<summary>Answer</summary>

**B. `VECTORPROPERTY(vector, 'Dimensions')`.**

</details>

### Question 6

A search must handle exact product identifiers and natural-language descriptions in the same query. Which design is most appropriate?

A. Vector search only  
B. Full-text search only  
C. Hybrid full-text and vector search  
D. Primary key search only

<details>
<summary>Answer</summary>

**C. Hybrid search.** Full-text retrieval preserves exact lexical matches, while vector retrieval supplies semantic matches. Their rankings can be combined with RRF.

</details>

### Question 7

Why should stored content and incoming queries use the same embedding model and configuration?

A. To make the SQL table smaller  
B. To place both in the same comparable vector space  
C. To enable full-text indexing  
D. To guarantee that ANN becomes exact

<details>
<summary>Answer</summary>

**B.** Vector distances are meaningful only when the vectors were generated in a compatible vector space.

</details>

### Question 8

Which statement about `VECTOR_DISTANCE` is correct?

A. It automatically uses a DiskANN index.  
B. It returns approximate results.  
C. It performs an exact distance calculation and does not use a vector index.  
D. It generates embeddings.

<details>
<summary>Answer</summary>

**C.** `VECTOR_DISTANCE` is exact, even when a vector index exists.

</details>

### Question 9

What does `VECTOR_NORMALIZE(@v, 'norm2')` do?

A. Changes the dimension count  
B. Produces a same-direction vector with Euclidean length 1  
C. Creates a vector index  
D. Calculates nearest neighbors

<details>
<summary>Answer</summary>

**B.** Normalization preserves direction while scaling magnitude according to the selected norm.

</details>

### Question 10

The source text for an existing row changes. What should happen to its embedding?

A. Nothing; embeddings never change.  
B. Regenerate it so it represents the updated source.  
C. Convert it to a full-text index.  
D. Only rename the vector column.

<details>
<summary>Answer</summary>

**B.** The embedding should be kept synchronized with the content it represents.

</details>

### Question 11

Which metric is usually a strong starting choice when the direction of text embeddings matters more than their magnitude?

A. Cosine distance  
B. Row count  
C. Edit distance  
D. Date difference

<details>
<summary>Answer</summary>

**A. Cosine distance.** It focuses on angular direction and is commonly used for semantic embeddings.

</details>

### Question 12

What is recall in ANN evaluation?

A. The number of dimensions in the embedding  
B. The proportion of true nearest neighbors found by the approximate search  
C. The time needed to generate an embedding  
D. The size of a SQL data page

<details>
<summary>Answer</summary>

**B.** Recall measures how much of the exact nearest-neighbor result set the ANN method successfully retrieves.

</details>

## 16. Rapid-review checklist

Before the exam, make sure you can explain each statement without notes:

- [ ] I can distinguish a vector from an embedding.
- [ ] I know why dimensions and embedding models must match.
- [ ] I can estimate `float32` vector storage.
- [ ] I can choose cosine, Euclidean, or dot distance from a scenario.
- [ ] I know that lower SQL distance values mean closer results.
- [ ] I can distinguish exact search from ANN.
- [ ] I know that `VECTOR_DISTANCE` is exact and does not use a vector index.
- [ ] I know that `VECTOR_SEARCH` is the approximate indexed-search feature.
- [ ] I know that SQL vector indexes use DiskANN.
- [ ] I can explain recall versus latency.
- [ ] I know the purposes of all five vector functions.
- [ ] I can choose full-text, vector, or hybrid search.
- [ ] I can describe a RAG ingestion and query flow.
- [ ] I understand chunk-size and overlap tradeoffs.
- [ ] I know why embeddings must be updated with their source data.
- [ ] I can identify security and model-lineage requirements.

## 17. Official Microsoft references

- [DP-800 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)
- [Vector search and vector indexes in the SQL Database Engine](https://learn.microsoft.com/en-us/sql/sql-server/ai/vectors?view=sql-server-ver17)
- [Vector data type](https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type?view=sql-server-ver17)
- [Vector functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-functions-transact-sql?view=sql-server-ver17)
- [`VECTOR_DISTANCE`](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql?view=sql-server-ver17)
- [`VECTOR_SEARCH`](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql?view=sql-server-ver17)
- [`VECTOR_NORMALIZE`](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-normalize-transact-sql?view=sql-server-ver17)
- [`VECTOR_NORM`](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-norm-transact-sql?view=sql-server-ver17)
- [`VECTORPROPERTY`](https://learn.microsoft.com/en-us/sql/t-sql/functions/vectorproperty-transact-sql?view=sql-server-ver17)
- [Design and implement intelligent search with SQL training module](https://learn.microsoft.com/en-us/training/modules/design-implement-intelligent-search-with-sql/)
- [Choose an Azure service for vector search](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/vector-search)

---

> **One-sentence exam summary:** Store model-compatible embeddings in a correctly sized `VECTOR` column, choose a suitable distance metric, use `VECTOR_DISTANCE` for exact retrieval or a DiskANN index with `VECTOR_SEARCH` for scalable ANN, and combine vector and full-text retrieval when both semantic meaning and exact terminology matter.
