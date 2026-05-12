---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: [concepts/splade-family-baselines.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 2
---

# Post-snapshot (splade-family-baselines): vector-scalar-bench-methodology

## TL;DR

无直接影响——sparse evolution chain 不涉及 scalar filter benchmark methodology. 但 NEW indirect: hybrid sparse+dense + scalar filter 三模 benchmark 现在 wiki 内有完整 sparse algorithm baseline 链 (BM25 → BlockMaxWAND → doc2query → DeepImpact → COIL → SPLADE)，**fair benchmark 时 sparse-side 不能再用泛指的 "BM25"——需 specify 是经典 lexical BM25 还是 neural sparse SPLADE-style**.

## Cited Pages

- [concepts/splade-family-baselines.md](../../concepts/splade-family-baselines.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
