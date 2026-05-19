---
query-key: index-architecture-global-vs-routed
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: [systems/cxl-anns.md, systems/distributedann.md, topics/disk-vs-memory-ann.md]
cited-count: 3
---

# Post-snapshot (jang-2023-cxl-anns): index-architecture-global-vs-routed

## TL;DR

**中等影响——为 (a) 全局单一索引 提供新论据**。CXL-ANNS 是 **single global NSG graph**（无路由层），但解决"全局太大装不下"的方式不是分片，而是**把单一图放进 CXL 解耦内存池**——架构上仍 (a) 全局单一索引，靠硬件 substrate 扩容而非逻辑分区。与 DistributedANN（single graph 跨 1000+ 机器 via KV store）形成"single-graph 派的两种 backend"：DistributedANN = distributed KV shared-disk；CXL-ANNS = CXL 解耦内存池。两者都用 near-data computation 削单一大图的内存带宽瓶颈。对比 (c) 层次路由（SPANN/Milvus segment）：CXL-ANNS 论点是路由层的 recall floor + 多跳网络开销可通过"不分区+解耦内存"绕开——但前提是有 CXL 硬件且单 cluster 内（CXL-ANNS multi-host 仅扩到 4 host，6 host PE 瓶颈）。

## Cited Pages

- [systems/cxl-anns.md](../../systems/cxl-anns.md)
- [systems/distributedann.md](../../systems/distributedann.md)（single-graph 派对比 backend）
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
