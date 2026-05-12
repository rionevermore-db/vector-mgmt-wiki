---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: [concepts/splade-family-baselines.md]
cited-count: 1
---

# Post-snapshot (splade-family-baselines): index-architecture-global-vs-routed

## TL;DR

间接 NEW: COIL (Gao 2021 NAACL) 提供**第三类拓扑**——per-token contextualized inverted list，**每 token 独立分片**而不是 doc-level 或 vector-level. 这意味着 hybrid retrieval at scale 不是简单 (a) 全局 vs (c) 路由 二选一——sparse-side 可以 token-partitioned (COIL/SPLADE)，dense-side 可以 vector-partitioned (IVF/HNSW)，**两套 partitioning axis 共存**. 关键 NEW: 千亿/万亿规模 hybrid 系统真实的拓扑选择是 **multi-axis partition** 而非 single hierarchy.

## Cited Pages

- [concepts/splade-family-baselines.md](../../concepts/splade-family-baselines.md)
