---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [concepts/rabitq.md, systems/spann.md, systems/diskann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 4
---

# Post-snapshot (gao-2024-rabitq): index-architecture-global-vs-routed

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 间接影响 (a) global 路径的可行性**——D-bit code 让 global IVF + RaBitQ 在单机 1 TB RAM 装千亿 vectors 理论可行（PQ 仅 ~600M）；这把"千亿必须 (c) 路由"的传统观点放宽——**单机大内存 + 极致量化** 是 (a) 的新可能。但生产实证仍 zero coverage——千亿 RaBitQ 全是 future。**(c) 内部 leaf-routing 接口**仍是 TopK black-box vs RM iterator 的 binary（per zhang-2023 post），RaBitQ 不直接影响这个轴。

## Answer

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| (a) global 单一索引 实证 | Faiss 1.5T mmap 是工业上限 | **+ RaBitQ 理论上单机千亿可行（D bits per vector）** |
| (c) 层次路由 实证 | 同前（SPANN + Bing / Milvus / Pinecone / Filtered-DiskANN） | **不变** |
| (c) 内部 leaf-routing 接口 | TopK black-box vs Iterator + RM | **不变** |
| Per-node quantizer | PQ 默认 | **+ RaBitQ alternative** |

### RaBitQ 对 (a) 路径的影响（NEW）

[per concepts/rabitq.md "Code length"]

768-d vector，1 TB RAM 单机：

| Quantizer | Code 长度 | 单机容量 | (a) 千亿可行性 |
|---|---|---|---|
| Flat 32-bit | 24 KB/vec | ~40M vectors | ✗ |
| PQ default | 1.5 KB/vec | ~600M vectors | ✗ (千亿差 ~150×) |
| PQ aggressive | 64 bytes/vec | ~15B vectors | 仍不到千亿 |
| **RaBitQ default** | **96 bytes/vec** | **~10B vectors** | **百亿可行，千亿仍差 10×** |
| Hypothetical RaBitQ + extreme padding (D/2 bits) | 48 bytes/vec | ~20B vectors | 千亿差 5× |

→ RaBitQ 让单机百亿可行（PQ ~15B aggressive 也勉强），但千亿仍需 (c) 路由——**未根本改变结论**。

但**多机 (a) global** 路径变得更现实：
- 16 节点 × 1 TB RAM × RaBitQ default = 16 × 10B = **160B vectors 总容量**——能装千亿
- 比 (c) 层次路由 storage overhead 更低（无 closure replication，无 partition centroids）
- 但 query 时仍需广播到所有 16 节点（无 routing 优化）→ latency / throughput trade-off

→ **多机 + RaBitQ + global IVF** 是理论上**新形态**（既不是 single-node global 也不是 SPANN-style routing），wiki 内 zero coverage。

### (c) 层次路由的"叶-根接口"细化（不变 from zhang-2023-vbase post）

```
路由 query
   ↓ (按 cluster 距离 / partition pruning 选 P partitions)
   ┌────────────────────────────────────────────┐
   │ for each partition p in P:                │
   │   Option 1 (legacy): leaf_p.topk(query, K')│
   │   Option 2 (VBASE): leaf_p iterator + RM   │
   └────────────────────────────────────────────┘
   ↓
返回 top-K
```

**RaBitQ 与两条 leaf-routing 接口都正交**——leaf 内可以用 RaBitQ 替代 PQ，不影响 root-leaf interface 形态。

### 三轴 trade-off 完整版（NEW）

| 决策 | (a) global | (c) routed | (a)/(c) 分界点 | RaBitQ 影响 |
|---|---|---|---|---|
| Storage | 简单 single index | 路由 + 多 leaf | 容量约束 | **RaBitQ 提升 (a) 容量上限** |
| Build | 单次 train | 多 leaf train | 复杂度约束 | RaBitQ index time = PQ |
| Query | 全扫 / 全广播 | 选 P partition | latency 约束 | RaBitQ 估计速度 3× faster than PQ |
| Update | 全 rebuild | 局部 leaf rebuild | 频率约束 | RaBitQ rebuild 与 PQ 同 |
| Filter | n/a | partition-based 优化 | 高频 filter 约束 | RaBitQ 与 filter 正交 |

→ RaBitQ 主要影响 storage 维度（让 (a) global 在更大 scale 仍可行）和 query 维度（更快距离估计）；不影响 (c) routing 哲学。

### 在 16 节点 × 1 TB 私有云的具体决策（不变）

仍是 (c) 层次路由——SPANN-style partition + DRAM/SSD 混合是工业实证唯一答案。

但**潜在 (a) global 多机**作为 future option：
- 16 节点 × RaBitQ + IVF + 全广播 query
- 千亿可行，但 P99 latency 受全广播限制
- 适合"读极少 / batch 离线"场景而非 online ANN

### 已知盲区

- **(a) + RaBitQ + 千亿实证**：完全空白
- **(c) + RaBitQ leaf**：完全空白（RaBitQ 单实例）
- **HCPS + (c) + RaBitQ**：完全空白
- **VBASE iterator + RaBitQ leaf**：完全空白（dual K' 攻击在 (c) 路由中的体现）

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
