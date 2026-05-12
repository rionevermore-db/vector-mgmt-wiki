---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md]
cited-count: 1
---

# Post-snapshot (frontier-2025-distributed-vector-search): hnsw-vs-nsg-selection

## TL;DR

间接 NEW: SPIRE paper §1 给出**实测**——HNSW 在分布式 (5-node) sharded 部署时 **>80% search steps 是 cross-node**, p99 latency 上升 2 个数量级. 即 HNSW 单机优势在分布式被 dense connectivity 反噬, NSG (sparser graph) **理论上 cross-node steps 应少**, 但 paper 未直接 benchmark. 关键 NEW: HNSW vs NSG **选型 axis 增加 "分布式部署 cross-node penalty"**——单节点选型 (HNSW) 与多节点选型 (可能更倾向 NSG 或非 dense graph) 是不同问题. SPIRE 自身用 proximity graph 但**只在 top level**, 中下层是 IVF-style partition——多种 graph 在 hierarchy 不同层级用.

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
