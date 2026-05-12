---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md]
cited-count: 1
---

# Post-snapshot (ding-2011-blockmaxwand): index-architecture-global-vs-routed

## TL;DR

BMW BM25 走 (c) hierarchical routing 的 sparse 特化: doc-ID hash sharding + per-shard inverted index + parallel fanout + per-shard BMW pruning. 与 wiki 8 dense architecture variant orthogonal.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
