---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md]
cited-count: 1
---

# Post-snapshot (chen-2024-bge-m3): memory-vs-disk-large-scale

## TL;DR

BGE-M3 三 output disk path 异质: dense 走 vector ANN (memory/SSD), sparse 走 inverted index (sequential SSD-friendly), colbert 走 token-level inverted file. 关键 NEW: production hybrid 在 disk-resident scenario 需要**3 path 各自 optimal disk layout**——sparse 50 年成熟 inverted index 优势在 BGE-M3 上发挥, dense + colbert 仍依赖各自 disk path.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
