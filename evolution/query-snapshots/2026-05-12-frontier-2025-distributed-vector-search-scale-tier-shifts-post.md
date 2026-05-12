---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md]
cited-count: 1
---

# Post-snapshot (frontier-2025-distributed-vector-search): scale-tier-shifts

## TL;DR

**NEW** (重要 inflection point insight): SPIRE 给出 "**balanced partition granularity inflection point**"——partition density 太 high 时 read amplification dominates, 太 low 时 cross-node steps dominates, 中间存在最优. 这是 wiki 第一次有 source-backed **量化分析** partition density 与 scale 的 trade-off. 关键 NEW:
1. **10 亿规模**: 单节点 RAM, partition granularity 影响小
2. **百亿规模**: 多节点 (10-30 节点), balanced granularity 开始重要
3. **千亿规模**: 必须 hierarchical, SPIRE recursive 多层端到端精度优化
4. **万亿规模**: SPIRE root level 自身需 sharding (8B 是 paper 上限, 万亿是开放问题)

关键 NEW 质变点: **百亿 → 千亿 partition granularity 设计 = 新质变档**——之前 wiki 质变描述基于"内存 → 磁盘"或"全局 → 路由", SPIRE 加入 partition 内 density 调优作为独立 axis.

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
