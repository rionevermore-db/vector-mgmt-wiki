---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: [concepts/disk-distributed-ann-2025-extensions.md]
cited-count: 1
---

# Post-snapshot (disk-distributed-ann-2025-extensions): scale-tier-shifts

## TL;DR

NEW: BatANN 在 1B 仍 viable, 与 SPIRE 8B 形成**单节点 vs 多节点 scale 跨度**:
- **10 亿 (1B)**: BatANN 10 server 配置仍 <6ms 均延迟 → 此规模 single global graph + baton-passing 是 viable 路径
- **百亿 (10B-100B)**: SPIRE 8B/46 节点验证, 实测 partition-routing hierarchy + balanced granularity 更优
- **千亿+ (100B+)**: 仍是 open——BatANN 上限是 1B (paper 验证), SPIRE 上限是 8B, 100B+ 是 DistributedANN 私有数据点 (Meta 1.5T)

关键质变点: **10 亿 → 百亿 = single global graph viability 上限**——超过 ~10B 必须 partition. BatANN 的 baton-passing 仅在 single graph viable 时有效, 是 single-graph paradigm 的延寿策略而非新质变.

## Cited Pages

- [concepts/disk-distributed-ann-2025-extensions.md](../../concepts/disk-distributed-ann-2025-extensions.md)
