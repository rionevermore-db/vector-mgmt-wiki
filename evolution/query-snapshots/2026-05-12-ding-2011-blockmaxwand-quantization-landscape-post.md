---
query-key: quantization-landscape
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md, concepts/splade-sparse-retrieval.md]
cited-count: 2
---

# Post-snapshot (ding-2011-blockmaxwand): quantization-landscape

## TL;DR

BlockMaxWAND 自身不是 quantization, 是 BM25 inverted index pruning. 但 BMW + 现代 sparse term weight quantization (e.g., SPLADE term impact int8 / int4) 联合 是 production sparse-side compression 主路径. 关键 NEW: sparse-side quantization 在 wiki 内 zero coverage 之前; BMW ingest 让 sparse-side quantization frontier 开始形成.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
