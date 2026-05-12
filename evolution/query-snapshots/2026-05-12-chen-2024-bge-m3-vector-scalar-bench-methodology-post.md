---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 2
---

# Post-snapshot (chen-2024-bge-m3): vector-scalar-bench-methodology

## TL;DR

BGE-M3 是 production embedding model 标准化候选——fair vendor benchmark 可用 BGE-M3 作 control variable (统一 embedding model + 多 vendor 实测 retrieval system performance). 关键 NEW: BGE-M3 + filter joint workload benchmark 在 vendor 端 zero coverage——但 BGE-M3 三 output 让 hybrid + filter benchmark methodology 更 nuanced (filter on each retrieval mode 各自评估).

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
