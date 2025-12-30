---
layout: single
title: "FusionRAG Just Got Simpler: BM25 is Now in PostgreSQL"
date: 2025-12-30
categories: [blogging]
---

In my [previous post about building a Local Knowledge Base MCP Server](/2025-10-28-MCP_RAG), I landed on Fusion RAG (BM25 + Vector) as the winning pattern. It caught both keywords and semantics, hitting 100% recall at 23ms.

The stack was: ChromaDB for vectors, rank-bm25 Python library for keyword search, custom fusion logic to merge results.

That stack just became simpler.  I discovered two PostgreSQL extensions can handle both pieces.

## The Old Architecture

```
┌─────────────┐     ┌─────────────┐
│  ChromaDB   │     │  rank-bm25  │
│  (Vectors)  │     │  (Keywords) │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └───────┬───────────┘
               │
        ┌──────▼──────┐
        │ Python Code │
        │ (Fusion)    │
        └─────────────┘
```

Two data stores. Sync issues. Custom fusion logic. Works, but more moving parts than necessary.

## The New Architecture

```
┌────────────────────────────────┐
│          PostgreSQL            │
│  ┌──────────┐  ┌────────────┐  │
│  │ pgvector │  │ pg_search  │  │
│  │ (Vectors)│  │   (BM25)   │  │
│  └──────────┘  └────────────┘  │
│         ┌──────────┐           │
│         │   RRF    │           │
│         │  (SQL)   │           │
│         └──────────┘           │
└────────────────────────────────┘
```

One database. One source of truth. Fusion happens in SQL.

## Why BM25 Matters

Standard PostgreSQL full-text search (ts_vector) does boolean matching: document either matches or it doesn't. No ranking. No relevance scoring.

BM25 solves four problems:

- **Term Frequency Saturation**: Mentioning a word 12 times doesn't make a doc 12x more relevant. After a few mentions, additional repetitions barely help.
- **Inverse Document Frequency**: Rare terms get higher weight. "Kubernetes" in a general corpus signals more than "the."
- **Length Normalization**: A focused 15-word answer beats a 80-word doc that mentions your query in passing.
- **Ranked Retrieval**: Every result gets a meaningful score, not just match/no-match.

BM25 is the algorithm powering Elasticsearch and Apache Lucene. 

## The Extensions

These are not built into PostgreSQL core. They are extensions you install separately.

[**pgvector**](https://github.com/pgvector/pgvector): First released April 2021. Adds vector data types and similarity search operators. HNSW indexing (the fast one) arrived in v0.5.0 (August 2023). Now at v0.8.x with broad cloud provider support.

[**pg_search**](https://github.com/paradedb/paradedb): First stable release November 2023 (originally called pg_bm25). Built on [Tantivy](https://github.com/quickwit-oss/tantivy), the Rust alternative to Lucene. Adds BM25 scoring and full-text search operators.

### pgvector vs ChromaDB

Here's how they compare:

| Aspect | ChromaDB | pgvector |
|--------|----------|----------|
| **Type** | Standalone vector database | PostgreSQL extension |
| **Best for** | Prototyping, up in 5 minutes | Production, existing PostgreSQL stack |
| **Concurrency** | Degrades under load | Handles concurrent queries well |
| **SQL joins** | Separate data store, needs sync | Native joins with relational data |
| **ACID** | No | Full transactions |
| **Scaling** | Purpose-built for vectors | PostgreSQL scaling patterns |

ChromaDB excels at rapid prototyping. Single queries are fast. But under concurrent load, pgvector tends to handle it better due to PostgreSQL's mature connection pooling and query optimization.

If you already run PostgreSQL, you eliminate a separate data store. User metadata, document content, and embeddings live in one place. One backup. One connection pool. No sync logic.

## Setting It Up

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_search;
```

Create your table with both vector and text columns:

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);
```

Create both indexes:

```sql
-- Vector index (HNSW for fast approximate search)
CREATE INDEX idx_docs_vector ON documents
USING hnsw (embedding vector_cosine_ops);

-- BM25 index
CREATE INDEX idx_docs_bm25 ON documents
USING bm25 (id, content)
WITH (key_field='id');
```

## Hybrid Search in Pure SQL

Reciprocal Rank Fusion (RRF) in a single query:

```sql
WITH
bm25_results AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY pdb.score(id) DESC) AS rank
  FROM documents
  WHERE content ||| 'kubernetes deployment strategy'
  LIMIT 20
),

vector_results AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> $1) AS rank
  FROM documents
  ORDER BY embedding <=> $1
  LIMIT 20
),

fused AS (
  SELECT id, 1.0 / (60 + rank) AS score FROM bm25_results
  UNION ALL
  SELECT id, 1.0 / (60 + rank) AS score FROM vector_results
)

SELECT
  d.id,
  d.content,
  SUM(f.score) AS relevance
FROM fused f
JOIN documents d USING (id)
GROUP BY d.id, d.content
ORDER BY relevance DESC
LIMIT 10;
```

The magic number 60 in RRF controls score decay. Lower values favor top results more aggressively.

## Weighted Fusion

In my MCP server, I used 30% keywords, 70% semantics. Same thing in SQL:

```sql
fused AS (
  SELECT id, 0.3 * 1.0 / (60 + rank) AS score FROM bm25_results
  UNION ALL
  SELECT id, 0.7 * 1.0 / (60 + rank) AS score FROM vector_results
)
```

Tune based on your data. Technical documentation with exact terms? Bump BM25 weight. Conversational queries? Favor vectors.

## What This Replaces

| Before | After |
|--------|-------|
| ChromaDB | pgvector |
| rank-bm25 (Python) | pg_search |
| Custom fusion code | SQL CTE |
| Two data stores | One database |
| Sync logic | ACID transactions |

## Operational Simplicity

Fewer dependencies, but also:

- **Backups**: One database to backup.
- **Consistency**: ACID transactions across text and vectors.
- **Scaling**: PostgreSQL scaling patterns you already know.
- **Monitoring**: One set of metrics.

When your documents update, both indexes update atomically. No sync jobs. No eventual consistency headaches.

## When to Still Use Elasticsearch

The 1% cases:

- Multi-petabyte scale with sub-100ms requirements
- Complex faceted search with dozens of filters
- Geo-spatial + full-text + vector in the same query at massive scale

For the rest of us building RAG pipelines, knowledge bases, and semantic search? PostgreSQL handles it.

## Trying It Out

Easiest path: [ParadeDB Docker image](https://hub.docker.com/r/paradedb/paradedb) comes with both extensions pre-installed.

```bash
docker run --name paradedb -e POSTGRES_PASSWORD=password -p 5432:5432 paradedb/paradedb
```

## Next Step for the MCP Server

The [Knowledge Base MCP Server](https://github.com/j3r3myfoobar/knowledge_base_mcp) currently uses ChromaDB + rank-bm25. Migrating to PostgreSQL would:

1. Remove two dependencies (chromadb, rank-bm25)
2. Simplify deployment (just needs a PostgreSQL connection)
3. Enable SQL-based analytics on search patterns
4. Make it easier to integrate with existing enterprise databases

The fusion logic moves from Python to a SQL view. The MCP server becomes a thin query layer.

Same accuracy. Simpler stack.
