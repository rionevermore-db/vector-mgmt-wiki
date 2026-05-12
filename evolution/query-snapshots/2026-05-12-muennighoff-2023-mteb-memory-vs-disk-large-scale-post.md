---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 1
---

# Post-snapshot (muennighoff-2023-mteb): memory-vs-disk-large-scale

## TL;DR

MTEB 不评估 memory-vs-disk path. 关键 NEW: MTEB-evaluated embedding 在 disk-resident vector DB (SPANN / DiskANN / Turbopuffer SPFresh / Chroma Cloud / 等) 下 quality 退化曲线 **wiki + 论文都 zero coverage**——这是 embedding-side benchmark (MTEB) 与 system-side disk path 跨 frontier gap.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
