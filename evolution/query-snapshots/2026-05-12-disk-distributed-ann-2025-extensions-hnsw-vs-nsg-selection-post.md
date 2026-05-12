---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: [concepts/disk-distributed-ann-2025-extensions.md]
cited-count: 1
---

# Post-snapshot (disk-distributed-ann-2025-extensions): hnsw-vs-nsg-selection

## TL;DR

间接 NEW: Gorgeous insight = "**graph structure access >> vector access**" 在 graph-based ANN 通用——意味着 HNSW (denser graph 上层) 与 NSG (sparser graph) 在 disk-resident 部署时 cache 行为差异更显著. HNSW 上层稀疏 + 下层密集的 hierarchical 结构在 Gorgeous 优化下尤其受益 (上层 graph 反复访问→cache hot). NSG 单 layer 但 sparser graph 总 cache 占用更小.

关键 NEW: HNSW vs NSG 选型新 axis = **disk 部署 cache efficiency**, 与 in-memory 选型 (HNSW 通常胜出) 不同——disk + Gorgeous-style layout 时两者 trade-off 接近.

## Cited Pages

- [concepts/disk-distributed-ann-2025-extensions.md](../../concepts/disk-distributed-ann-2025-extensions.md)
