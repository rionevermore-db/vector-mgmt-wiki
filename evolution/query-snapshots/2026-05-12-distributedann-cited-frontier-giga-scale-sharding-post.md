---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: distributedann-cited-frontier
wiki-pages-total: 83
cited-pages: [concepts/distributedann-cited-frontier.md, systems/distributedann.md]
cited-count: 2
---

# Post-snapshot (distributedann-cited-frontier): giga-scale-sharding

## TL;DR

4 paper 都 billion-scale ANN 路径 candidates. 关键 NEW: Gottesbüren 2024 是 **DistributedANN single-graph distributed 的直接对比 partitioning approach**——DistributedANN paper §4.4 admits "partitioning approach preferable in latency-constrained scenarios". CXL-ANNS / LM-DiskANN / AiSAQ 是 single-node giga-scale alternative paths. 千亿规模选择 matrix 更 complete: distributed graph (DistributedANN) vs distributed partitioning (Gottesbüren) vs CXL-disaggregated single-node (CXL-ANNS) vs DRAM-free single-node (AiSAQ / LM-DiskANN).

## Cited Pages

- [concepts/distributedann-cited-frontier.md](../../concepts/distributedann-cited-frontier.md)
- [systems/distributedann.md](../../systems/distributedann.md)
