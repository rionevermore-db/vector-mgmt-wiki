---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: thakur-2021-beir
wiki-pages-total: 85
cited-pages: [benchmarks/beir-heterogeneous-zero-shot-ir.md]
cited-count: 1
---

# Post-snapshot (thakur-2021-beir): memory-vs-disk-large-scale

## TL;DR

BEIR 主要 in-memory eval, 不评估 disk path performance. 但 BM25-robust finding 让 sparse-path (inverted index disk-friendly) 在 OOD setting 重要性提升——大规模 disk-resident workload 应保留 BM25 路径.

## Cited Pages

- [benchmarks/beir-heterogeneous-zero-shot-ir.md](../../benchmarks/beir-heterogeneous-zero-shot-ir.md)
