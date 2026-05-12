---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md]
cited-count: 1
---

# Post-snapshot (ding-2011-blockmaxwand): giga-scale-sharding

## TL;DR

BlockMaxWAND 是 inverted index query algorithm, sharding 自然走 inverted index 主流 (doc-ID hash sharding + per-shard inverted index + parallel fanout). 与 dense ANN sharding (5 architecture variant) 不同 axis——BMW BM25 在 giga-scale 线性 scale 不需要复杂 routing.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
