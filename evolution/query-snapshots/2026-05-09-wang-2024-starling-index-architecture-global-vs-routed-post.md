---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, systems/milvus.md, concepts/block-shuffling.md, topics/disk-vs-memory-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (wang-2024-starling): index-architecture-global-vs-routed

## TL;DR (delta from gao-2024-rabitq post)

**Starling 给 (c) 层次路由结构添加新的"叶节点"形态**——之前 (c) 的叶 (leaf) 是 single-server disk index（DiskANN/SPANN）；Starling 揭示叶可以是 **per-segment disk graph + block shuffling layout + in-memory navigation graph**。这把"路由 → 叶"二层结构细化为"路由 → 节点 → segment → block"四层：每一层都有独立优化空间。Starling §6.11 的 BIGANN 1B = 31 segments × 2 query nodes 是这个细化结构的**首个 billion-scale 实证**。**对 (a) global 路径的影响**：仍不可行（千亿单实例不现实），但 Starling 让 (c) 路由的"叶端"性能极大提升——更可能维持 (c) 优势。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| (a) global 单一索引 | 不变 | 不变（Starling 仍是 segment-level） |
| (c) 层次路由 | 同前 | **+ 叶节点形态细化（per-segment Starling）** |
| (c) 内部 leaf-routing 接口 | TopK black-box vs Iterator + RM | 不变 |
| **Per-segment disk index 选择** | DiskANN / Disk-NSG / Disk-HNSW | **+ Starling-Vamana / NSG / HNSW（2× 各自 baseline）** |
| Production 实证 | (c) 主流 | **+ Starling BIGANN 1B 31 segments** |

### (c) 层次路由的"四层结构"细化（NEW）

[per systems/starling.md "Scale 边界" + queries/index-architecture-global-vs-routed.md]

之前 wiki 把 (c) 层次路由理解为**两层**："root routing + leaf disk index"：

```
路由 query
   ↓
   ┌──────────────────────────────────┐
   │ root: routing layer              │
   │   - select P partitions          │
   │   - 一般是 IVF centroids       │
   └─────────────┬────────────────────┘
                 ↓
   ┌──────────────────────────────────┐
   │ leaf: disk index per partition   │
   │   - DiskANN / SPANN              │
   │   - single-server budget         │
   └──────────────────────────────────┘
```

Starling 揭示更细的**四层结构**：

```
路由 query
   ↓
   ┌──────────────────────────────────────────┐
   │ Layer 1: Cluster routing                  │
   │   - select query nodes                    │
   └─────────────────┬────────────────────────┘
                     ↓
   ┌──────────────────────────────────────────┐
   │ Layer 2: Node-level routing               │
   │   - select segments within node           │
   │   - 1 TB RAM 多 segment 共享             │
   └─────────────────┬────────────────────────┘
                     ↓
   ┌──────────────────────────────────────────┐
   │ Layer 3: Segment-level disk graph         │
   │   - in-memory navigation graph (sample)   │
   │   - reordered disk graph (block shuffle)  │
   └─────────────────┬────────────────────────┘
                     ↓
   ┌──────────────────────────────────────────┐
   │ Layer 4: Block-level data locality        │
   │   - 4KB block, 16 vertex/block            │
   │   - OR(G) 0.34-0.87 (vs DiskANN 0)        │
   │   - block search + pruning + pipeline     │
   └──────────────────────────────────────────┘
```

→ **每一层都有独立优化空间** + 独立失败模式：
- Layer 1: cluster 数量与跨网络 latency
- Layer 2: per-node segment 数量与 RAM/disk 共享
- Layer 3: in-memory graph quality + disk graph 算法选择（Vamana / NSG / HNSW）
- Layer 4: block shuffling NP-hard 优化 + block search 策略

Starling 主要 attack Layer 3-4；前两层是 vector DBMS engineering（Milvus segment 模型）。

### (a) vs (c) 的更新边界（NEW）

| 决策 | (a) global 单一索引 | (c) 层次路由 | Starling 影响 |
|---|---|---|---|
| 千亿可行性 | ✗（单实例 RAM 不够） | ✓（实证） | 不变 |
| 索引性能 | n/a | DiskANN 慢, SPANN 不可行 | **Starling 让 leaf 性能 2-43.9× DiskANN** |
| Build time | 5+ 天（DiskANN 1B） | per-segment parallel build | Starling per-segment build ≈ DiskANN segment build |
| Update 灵活度 | 整索引 rebuild | per-segment rebuild | 不变 |
| Fault tolerance | 跨 node replicate | **per-segment replicate** | Starling 加强 segment-level 路径 |

→ Starling **不挑战 (a)/(c) 的根本选择**——千亿仍必须 (c)。但 Starling 让 (c) 的叶端"性能下限"显著上升——之前 (c) 的痛点（叶端 disk graph 慢）被 attack。

### Starling 的论文 §1 Fig 1 启示（NEW）

[wang-2024-starling §1 Fig 1]

论文 Fig 1 直接对比两种 indexing strategy on single machine：
- **Strategy I**：one big index per machine（"too large"标识；不可行）
- **Strategy II**：multiple segments per machine（绿色 ✓）

→ 论文 explicit 反对 single-machine global index——这与 wiki 内 (a) global 路径的传统讨论形成 echo。

### 在 16 节点 × 1TB RAM 私有云的具体决策（不变）

[per queries/index-architecture-global-vs-routed.md]

(c) 层次路由 + per-machine 多 segment + Starling per-segment——是 production 唯一可行答案。

Starling 不改变这个答案，**但显著提升叶端性能**——之前用 DiskANN 的 segment 现在用 Starling 后 ANNS 2× / RS 43.9×。

### 已知盲区

- **多 node + query coordinator** Layer 1 routing：完全空白
- **跨 segment 优化**：Starling 单 segment 内最优；跨 segment vector 关系（cross-segment NN）未利用
- **Layer 1-4 联合优化**：每层独立调优 vs 联合优化 trade-off 未实证
- **Pinecone serverless slabs vs Starling**：Pinecone serverless 用 slab + adaptive indexing，与 Starling segment + block shuffling 是不同抽象——比较未做
- **HCPS + (c) + Starling**：完全空白

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/block-shuffling.md](../../concepts/block-shuffling.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
