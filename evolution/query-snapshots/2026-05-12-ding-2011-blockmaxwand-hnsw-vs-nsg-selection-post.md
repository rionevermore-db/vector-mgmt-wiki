---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md, concepts/hnsw.md]
cited-count: 2
---

# Post-snapshot (ding-2011-blockmaxwand): hnsw-vs-nsg-selection

## TL;DR

BlockMaxWAND 是 sparse-side BM25 retrieval algorithm, 与 HNSW/NSG (dense ANN graph) 是不同 axis 的 production retrieval algorithm. Wiki HNSW production OSS deployment 仍 7 vendor (Milvus + Qdrant + Weaviate + Vespa + pgvector + Chroma + LanceDB); BlockMaxWAND BM25 在 5+ vendor 是 sparse path default optimization.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
