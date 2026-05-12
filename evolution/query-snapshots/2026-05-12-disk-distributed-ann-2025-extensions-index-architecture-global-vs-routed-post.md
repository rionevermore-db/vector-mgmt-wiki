---
query-key: index-architecture-global-vs-routed
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: [concepts/disk-distributed-ann-2025-extensions.md, concepts/frontier-2025-distributed-vector-search.md]
cited-count: 2
---

# Post-snapshot (disk-distributed-ann-2025-extensions): index-architecture-global-vs-routed

## TL;DR

**重大 NEW** (直接对该 query): BatANN 给出 **(a) 全局单一索引 + baton-passing** 路线 = **新 viable 第四类**:
- 之前 wiki 内 3 类:
  1. (a) 全局单一 + scatter-gather (传统, 高 RPC 不可生产用)
  2. (c) 层次路由 (DSPANN/Pinecone-pod)  
  3. (c') Accuracy-preserving multi-level (SPIRE)
- 新增 **(a') 全局单一 + baton-passing query state migration** (BatANN, Ingest #15)

(a') 与 (c)/(c') 形成 **2025 末分布式 ANN 设计哲学二选一**:
- 保 single global graph + 高 fidelity + 跨 node query state 迁移 (BatANN) vs
- 弃 single graph + balanced hierarchical + end-to-end accuracy preservation (SPIRE)

千亿/万亿主流取决于 workload heterogeneity——同质 query batch + 高 bandwidth network → (a') BatANN; 异质 batch + 一般 network → (c') SPIRE.

## Cited Pages

- [concepts/disk-distributed-ann-2025-extensions.md](../../concepts/disk-distributed-ann-2025-extensions.md)
- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
