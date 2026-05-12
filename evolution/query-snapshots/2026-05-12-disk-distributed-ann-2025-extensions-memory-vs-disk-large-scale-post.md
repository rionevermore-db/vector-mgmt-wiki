---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: [concepts/disk-distributed-ann-2025-extensions.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (disk-distributed-ann-2025-extensions): memory-vs-disk-large-scale

## TL;DR

**重大 NEW**: 3 paper 各自填 wiki disk-based ANN 不同空白:
1. **Gorgeous = 第 11 类 disk philosophy axis**: "graph structure vs vector data 缓存优先级" 独立 axis. 之前 10 类聚焦"vector 放哪", Gorgeous 引入"图 vs 向量谁 cache 优先".
2. **BatANN = 单节点 disk → 多节点 disk 桥接**: 之前 wiki disk-based ANN 主要单节点 (DiskANN/Vamana/Starling) 或粗粒度 hierarchical (SPANN). BatANN 是 wiki 内**第一个 OSS 学术 multi-node disk-based with single global graph**, 是 1B → 100B 规模过渡的关键 paper.
3. **SPI = disk-based ANN 之上的 VecDB 层 multi-resolution**: 与 storage tier 正交, 在数据"是否需要存多级 chunk-granularity index"维度补充.

关键 NEW: 千亿 production disk-based ANN 现在有**两条 OSS 路径** + Gorgeous-style 单节点优化叠加, 实现成本显著低于商用 (Pinecone / Weaviate cloud).

## Cited Pages

- [concepts/disk-distributed-ann-2025-extensions.md](../../concepts/disk-distributed-ann-2025-extensions.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
