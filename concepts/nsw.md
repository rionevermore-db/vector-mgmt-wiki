---
title: NSW（可导航小世界图）
type: concept
sources: [malkov-2016-hnsw]
related: [hnsw.md, proximity-graph.md, nsg.md]
created: 2026-04-30
updated: 2026-05-07
---

# NSW

**TL;DR**: 在线增量插入构造的 [proximity graph](./proximity-graph.md)，靠"先插入的元素自然变成 hub"形成长程链接，从而实现可导航小世界结构。HNSW 的直接前作。[malkov-2016-hnsw §2.2]

## 提出背景

Malkov、Ponomarenko、Logvinov、Krylov 提出 [25][26][30] in malkov-2016-hnsw。
目标：构造一个无需训练、无需全局协调、支持任意度量空间的 ANN 索引。

## 关键性质

- **构造方式**：元素按随机顺序逐个插入，对每个新元素用当前图做 K-NN 搜索（多 entry point 贪心），把它双向连到 M 个最近邻居。[malkov-2016-hnsw §2.2]
- **可导航性来源**：先插入的元素邻居稀疏（因为图小），后续元素加入后这些早期节点保留了大量"跨距离尺度"的连接，自然成为 hub。
- **搜索机制**：从随机节点出发贪心，先 zoom-out（从低度节点爬到高度 hub），再 zoom-in（从 hub 跳到目标邻域）。[malkov-2016-hnsw §3]
- **复杂度**：单次搜索 polylog——平均跳数 O(log N) × 路径上节点平均度 O(log N) ≈ O(log² N)。[malkov-2016-hnsw §3]
- **天然分布式**：构造和搜索都不需要全局同步，可拆到多机；是 NSW 相对 [HNSW](./hnsw.md) 的少数优势之一。[malkov-2016-hnsw §2.2, §6]

## 失败模式

1. **聚类数据上严重退化**——在低维聚类数据上比基于树的算法慢几个数量级 [34 in malkov-2016-hnsw]。原因：贪心可能停在簇边界的伪局部最优。
2. **polylog 而非 log**——搜索路径上节点度数随 N 增长，整体复杂度高于 skip-list 类结构。
3. **非度量空间崩溃**——在 wiki-8（JS 散度）数据集上几乎不可用，HNSW 在同数据集上快约 3 个数量级。[malkov-2016-hnsw §5.3 + Fig 14]

## 与 HNSW 的关键区别

| | NSW | HNSW |
|---|---|---|
| 图结构 | 单层 | 多层（指数衰减层级） |
| 长程链接来源 | 隐式（早期节点 hub 化） | 显式（顶层即长程） |
| Entry point | 随机或多个 | 固定（顶层最高 level 节点） |
| 邻居选择 | 选 M 个最近 | 启发式（Alg 4，保留跨簇边） |
| 复杂度 | polylog | log（经验） |
| 分布式 | 天然 | 困难 |

[malkov-2016-hnsw §3, §6]

## 典型实现

- nmslib 中的 `sw-graph`。HNSW 论文的 baseline 即此实现。[malkov-2016-hnsw §5]

## Open Questions

- NSW 在聚类数据上的退化是否能不靠分层、只靠改进邻居选择启发式（如 [HNSW](./hnsw.md) 的 Alg 4）来缓解？论文 §5.2 Fig 7 暗示部分可以——把 Alg 4 加到 NSW 上能恢复一些性能。
