---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: thakur-2021-beir
wiki-pages-total: 85
cited-pages: [benchmarks/beir-heterogeneous-zero-shot-ir.md]
cited-count: 1
---

# Post-snapshot (thakur-2021-beir): index-architecture-global-vs-routed

## TL;DR

BEIR 不涉及 sharding architecture (eval on individual retrieval systems). 但是 "BM25 + CE 是 best hybrid at high cost" finding 让 production architecture decision 包含 rerank stage——影响 Vespa 4-phase ranking / DistributedANN-style 选择.

## Cited Pages

- [benchmarks/beir-heterogeneous-zero-shot-ir.md](../../benchmarks/beir-heterogeneous-zero-shot-ir.md)
