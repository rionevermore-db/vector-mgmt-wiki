---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md]
cited-count: 1
---

# Post-snapshot (frontier-2025-distributed-vector-search): index-architecture-global-vs-routed

## TL;DR

**重大 NEW** (直接对该 query): SPIRE 实测两种拓扑 trade-off + 给出**第三类** "**accuracy-preserving multi-level**":
1. **(a) 全局单一索引** (HNSW sharded): SPIRE 实测 5-node 时 >80% search steps cross-node, p99 latency 上升 2 个数量级——**生产不可用**
2. **(c) 层次路由**: SPIRE 实测 DSPANN 8B index 需探 9/46 partitions for recall@5=0.9 due to **fidelity loss**——throughput 严重下降
3. **(c') Accuracy-preserving multi-level (SPIRE 新)**: 与 (c) 区别 = **递归构造每层时优化端到端 accuracy 而非 per-level**, 同时 balanced granularity 控制 partition density——相对 SOTA hierarchical 取得 9.64× peak throughput

关键 NEW: 千亿/万亿规模主流 = (c) 但 production 需用 (c') 改进版. **Per-level optimal ≠ end-to-end optimal** 是新设计原则.

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
