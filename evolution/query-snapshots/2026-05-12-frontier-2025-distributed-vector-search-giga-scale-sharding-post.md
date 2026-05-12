---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md, concepts/distributedann-cited-frontier.md]
cited-count: 2
---

# Post-snapshot (frontier-2025-distributed-vector-search): giga-scale-sharding

## TL;DR

**重大 NEW** (talk 主题直接相关): SPIRE (Xu et al. 2025 USTC + Microsoft Research) 给出 wiki **首个 8B vectors / 46 nodes / 9.64× SOTA throughput 实测数据点**——直接对应千亿/万亿规模 16-node + 1TB RAM 配置. 关键 NEW insight:
1. **End-to-end accuracy preservation** > per-level optimization——DSPANN / SPTAG / Pinecone-pod 都 per-level 优化, fidelity loss 跨层累积; SPIRE 推荐 recursive 多层端到端
2. **Balanced partition granularity** 是 sweet spot——不能太 dense (read amp) 也不能太 sparse (cross-node steps), 量化 inflection point 设计
3. **Stateless compute tier**: in-memory root replicated, SSD level easy reconstruct——千亿规模 elastic scaling 关键
4. 16-node 1TB RAM = 16TB RAM total, SPIRE 8B/46-node = 200M vec/node, 16 node 100B vec = 6.25B vec/node → 仍在 SPIRE 验证范围近邻

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
- [concepts/distributedann-cited-frontier.md](../../concepts/distributedann-cited-frontier.md)
