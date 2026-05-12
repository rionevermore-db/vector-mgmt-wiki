---
query-key: multimodal-bench-methodology
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md]
cited-count: 1
---

# Post-snapshot (ding-2011-blockmaxwand): multimodal-bench-methodology

## TL;DR

BMW 是 BM25 text-retrieval optimization, 不涉及 multimodal. 关键 NEW: production multimodal hybrid pipeline 中, text-side BM25 (BMW-optimized) 与 visual-side CLIP cross-modal 是不同 retrieval path. Vendor 需要 BMW (text BM25) + dense ANN (CLIP) 同时支持.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
