---
source-key: chroma-docs
title: Chroma 官方文档（llms-full.txt）
vendor: Chroma (Chroma Core + Chroma Cloud, Anton Troynikov / Jeff Huber 2022)
source-url: https://docs.trychroma.com
acquisition-method: llms-full-txt
fetched-at: 2026-05-12
files:
  - llms-full.txt   # 21716 行 / 810 KB — Chroma Cloud + OSS complete docs
version-coverage: Chroma Core OSS Apache-2.0 + Chroma Cloud (Distributed Chroma) — as of 2026-05-12
notes: |
  Chroma is RAG-dev-experience leader vector DB. OSS Apache-2.0 core +
  Chroma Cloud managed service. Founded by Anton Troynikov + Jeff Huber 2022.
  
  Key distinctions vs prior path C ingests:
    Milvus / Qdrant / Weaviate / Vespa: OSS DBMS production focus
    Pinecone: closed SaaS only, separate vector DBMS
    Turbopuffer: closed SaaS, object storage native, SPFresh
    pgvector: Postgres extension, ACID via Postgres
    Chroma: **OSS core + Cloud managed** + RAG dev-experience focus + MCP server
            integration for AI agents + SPANN-based Cloud + HNSW-based OSS
---

# Chroma Documentation Source

This directory holds the Chroma documentation acquired via llms-full.txt
endpoint on 2026-05-12.

## Acquisition

- Path C priority 1 (llms-full.txt) — hit
- 21716 lines / 810 KB

## Key sections

- Chroma Cloud overview / Distributed Chroma architecture
- Collection forking (copy-on-write, Chroma Cloud only)
- Package Search MCP Server (AI agent integration)
- Index Configuration Reference (6 index types)
- Schema Overview / Basics
- Sparse Vector Search Setup (SPLADE / BM25 / HuggingFace sparse)
- Batch Operations
- Filtering with Where
- Group By & Aggregation
- Hybrid Search with RRF
- Search API Overview (advanced ranking, batch operations)
- Pagination & Field Selection
- Pricing ($2.50/GiB write, $0.0075/TiB query, $0.33/GiB/month storage)
- Chroma Sync (S3 / GitHub / Web / file upload)
