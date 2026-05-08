---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [systems/vbase.md, systems/spann.md, concepts/relaxed-monotonicity.md, topics/topk-vs-iterator-model.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (zhang-2023-vbase): index-architecture-global-vs-routed

## TL;DR (delta from patel-2024-acorn post)

**(c) 层次路由结构的"路由-叶子"接口选择新增一轴：TopK black-box vs Iterator + RM**——VBASE+SPANN (§5.4) 实证 partition-based 索引（叶子）可以走 RM iterator 与 query engine（路由）通信，而不是传统的 black-box TopK。这意味着**(c) 内部的"path 通信抽象"不再唯一**——除了 routing layer 决定"扫哪些 partition"，还有**leaf 给 routing 返回什么形态**：固定 K 个 TopK 还是流式 RM iterator？后者让 cross-partition 的 NRA-style 合并变得可能（理论上）。

## Answer

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| (a) global 单一索引 实证 | 同前（Faiss 1.5T mmap） | **不变** |
| (c) 层次路由 实证 | 同前（SPANN + Bing / Milvus segment / Pinecone slabs / Filtered-DiskANN） | **不变** |
| (c) 内部 leaf-routing 接口 | 默认 TopK black-box | **+ Iterator + RM 选项（VBASE+SPANN 实证）** |
| Production 实证 | (c) 主流（千亿 / 万亿） | **不变**——VBASE+SPANN 学术 only |

### (c) 层次路由的"叶-根接口"细化（NEW）

[per topics/topk-vs-iterator-model.md + benchmarks/vbase-8queries-recipe1m.md]

之前 wiki 隐含假设：(c) routing → leaf 接口是 **TopK(K_per_partition)**：

```
路由 query
   ↓ (按 cluster 距离 / partition pruning 选 P partitions)
   ┌────────────────────────────────────────────┐
   │ for each partition p in P:                │
   │   results_p = leaf_p.topk(query, K')      │  ← 黑盒
   │ merged = merge(results_p across P)        │  ← K' 静态
   └────────────────────────────────────────────┘
   ↓
返回 top-K
```

**问题**：K' 必须静态预测——partition 内 selectivity 不可知。SPANN 的 query-aware dynamic pruning 缓解但仍是 K-based。

VBASE+SPANN 的新形态（**未实证**但理论上可行）：

```
路由 query
   ↓
   ┌────────────────────────────────────────────┐
   │ open iter_p for each partition p          │
   │ NRA-style merge:                          │
   │   while not all in Phase 2:               │
   │     pull next from leaf with smallest d   │
   │     accumulate top-K with global threshold│
   └────────────────────────────────────────────┘
   ↓
返回 top-K（自动停 with Phase 2）
```

→ **跨 partition NRA**——routing layer 用 RM 检测整体 Phase 2，自动决定 stop。**消除 cross-partition K' 选择**。

### 但当前实证是单 partition VBASE+SPANN（NEW）

[per zhang-2023-vbase §5.4 + Table 8]

VBASE+SPANN 实测是**单 partition**——SPANN 在单实例 PG 内作为 vector index。**未实证**：
- 跨多机 partition 的 RM 协调
- 跨 partition 的 NRA aggregation
- routing layer 是否暴露 RM 给应用

→ 这是 wiki 内**全新的开放问题**：**(c) 层次路由 + Iterator + RM** 的 distributed 形态。

### 在 16 节点 × 1TB 私有云的具体决策（不变）

[per queries/index-architecture-global-vs-routed.md]

- 千亿规模 → (c) 层次路由
- 16 节点 × 1TB RAM = 16 TB 内存预算
- 工业实证仍是 SPANN @ Bing / Faiss 1.5T / Milvus segment / Pinecone slabs

VBASE 不改变 storage / partition / routing 的工业实证。它**只在 query engine layer 提供新选项**——但需要 16 节点上各自集成 RM iterator + 中心 query engine 协调。

### 已知盲区

- **(c) + 跨 partition RM iterator 的工业实证**：完全空白
- **VBASE 单 partition 实证 vs 跨 partition theoretical**：gap 巨大
- **HCPS + (c) + Iterator**：完全空白
- **Pinecone serverless 是否实现类 RM 接口**：闭源未知（理论上 slab 演化式索引可能 internal 用类似抽象）

## Cited Pages

- [systems/vbase.md](../../systems/vbase.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/relaxed-monotonicity.md](../../concepts/relaxed-monotonicity.md)
- [topics/topk-vs-iterator-model.md](../../topics/topk-vs-iterator-model.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
