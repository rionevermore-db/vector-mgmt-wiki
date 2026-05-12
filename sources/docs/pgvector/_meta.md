---
source-key: pgvector-docs
title: pgvector 官方文档 (README.md)
vendor: pgvector (open-source community, primary maintainer Andrew Kane)
source-url: https://github.com/pgvector/pgvector
acquisition-method: github-readme-download
fetched-at: 2026-05-12
files:
  - README.md   # 1357 行 / 42 KB — canonical doc for pgvector extension
version-coverage: pgvector 0.8.2 (PostgreSQL 13+, master branch as of 2026-05-12)
notes: |
  pgvector is open-source Postgres extension for vector storage and similarity search,
  PostgreSQL License (similar to BSD). Maintained primarily by Andrew Kane (Ankane).
  Industry-most-deployed OSS vector option (Postgres extension path).
  
  Single README.md is canonical doc (no separate docs site).
  
  Compared to other path C ingests:
    Milvus (GitHub docs, 116 MB) — full OSS Go DBMS
    Qdrant (GitHub sparse-checkout, 11 MB) — full OSS Rust DBMS
    Weaviate (llms.txt + 5 twin pages) — OSS Go DBMS
    Vespa (llms-full.txt 92K lines) — OSS C++/Java engine
    Pinecone (llms-full.txt 87K lines) — closed SaaS
    Turbopuffer (llms-full.txt 15K lines) — closed SaaS
    pgvector — minimal 1.3K-line README serves as canonical doc (扩展 not DBMS)
  
  Position in wiki: 与 PASE (Ant Financial Postgres extension) 和 VBASE
  (Microsoft Research Postgres extension) 形成 "Postgres-extended vector
  retrieval" 三-source triangle. pgvector 是这三 RDBMS-extended 路径中
  industry deployment 最广的。
---

# pgvector Documentation Source

This directory holds the pgvector README.md acquired from GitHub on 2026-05-12.

## Acquisition

- GitHub README direct download (path C alternative since no llms.txt or
  separate docs site exists)
- README.md serves as canonical documentation
- 1357 lines / 42 KB

## Key sections

- Overview / Installation
- Vector Types: vector / halfvec / bit / sparsevec
- HNSW indexing: m, ef_construction, ef_search parameters
- IVFFlat indexing: lists, probes parameters
- Distance operators: `<->` L2 / `<#>` neg inner / `<=>` cosine / `<+>` L1 / `<~>` Hamming / `<%>` Jaccard
- Iterative index scans (v0.8.0+): strict_order / relaxed_order modes
- Filtering strategies (partial indexes, table partitioning)
- Hybrid search (PostgreSQL FTS + vector)
- Binary quantization (expression-based indexing)
- Performance tuning (maintenance_work_mem, parallel workers)
- Troubleshooting

## Notes on long-term stability

- pgvector evolves rapidly (0.8.2 as of 2026-05-12)
- Future re-fetches should go to sources/docs/pgvector-<YYYY-MM>/ to preserve history
- README on master branch may include unreleased features
