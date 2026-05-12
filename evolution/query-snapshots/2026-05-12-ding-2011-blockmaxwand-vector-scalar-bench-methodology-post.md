---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 2
---

# Post-snapshot (ding-2011-blockmaxwand): vector-scalar-bench-methodology

## TL;DR

BMW BM25 benchmark 公平比较通常 anchor on TREC / MS MARCO + Lucene 实现. 关键 NEW: vendor 间 BMW BM25 实现差异 (Lucene-derived vs Vespa 自研 vs Turbopuffer 自研) 在公平 benchmark 内是 confounding factor——需要 control BMW implementation version + posting list compression scheme.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
