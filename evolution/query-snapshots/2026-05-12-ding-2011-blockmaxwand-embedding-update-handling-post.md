---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md]
cited-count: 1
---

# Post-snapshot (ding-2011-blockmaxwand): embedding-update-handling

## TL;DR

BlockMaxWAND BM25 不依赖 embedding model——BM25 是 statistical (TF-IDF 后继), 不需要 neural model. **关键 NEW**: BMW BM25 path 是 production hybrid pipeline 的**"model-independent"路径**——neural embedding (sparse/dense/colbert) 升级时, BMW BM25 inverted index 不需 rebuild. 是 hybrid pipeline upgrade strategy 内 "stable axis" 的 vendor.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
