---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: []
cited-count: 0
---

# Post-snapshot (frontier-2025-distributed-vector-search): embedding-update-handling

## TL;DR

无直接影响——3 paper 都假设 embedding model 固定. 间接: Trinity 的 vector search pool 完全 decoupled 意味着 **embedding model 升级时, prefill/decode pool 不动, 仅 vector search pool 重建**——cleaner separation; 但仍需全量 re-encode + 重 build vector index. SPIRE recursive 构造 cost 在升级时仍 dominant.

## Cited Pages

(无)
