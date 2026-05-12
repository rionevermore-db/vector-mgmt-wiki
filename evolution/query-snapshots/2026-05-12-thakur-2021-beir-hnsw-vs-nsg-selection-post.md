---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: thakur-2021-beir
wiki-pages-total: 85
cited-pages: [benchmarks/beir-heterogeneous-zero-shot-ir.md]
cited-count: 1
---

# Post-snapshot (thakur-2021-beir): hnsw-vs-nsg-selection

## TL;DR

BEIR 评估 retrieval system 不评估 ANN algorithm. HNSW production OSS deployment 仍 7 vendor 不变. 但 BEIR 关键 finding "BM25 robust baseline in OOD setting"——production hybrid retrieval pipeline 应保留 BM25 path 作 dense retrieval 的 OOD safety net.

## Cited Pages

- [benchmarks/beir-heterogeneous-zero-shot-ir.md](../../benchmarks/beir-heterogeneous-zero-shot-ir.md)
