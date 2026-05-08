---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/rabitq.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]
cited-count: 4
---

# Post-snapshot (gao-2024-rabitq): hnsw-vs-nsg-selection

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 不直接影响 HNSW vs NSG 选择**——RaBitQ 与 graph-based 索引集成开放（论文 §4 明示 future work）。但 RaBitQ + IVF 实测 **6/6 dataset dominate HNSW**——这给"single-vector TopK 是否一定选 graph"的传统答案打上 question mark：在 memory budget 严的场景，IVF + RaBitQ 可能比 HNSW 更优。HNSW vs NSG 之间的相对 trade-off 不变，但**两者一起作为"graph path"** 与 **"IVF + RaBitQ path"**之间的边界发生迁移。

## Answer

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| HNSW 集成进展 | + iterator + RM (VBASE) | **不变**（RaBitQ 与 graph 集成开放） |
| NSG 集成进展 | + iterator + RM 理论可行未实证 | **不变** |
| HNSW vs NSG 工程取舍 | + "是否 unified query engine" 新轴 | **+ "vs IVF + RaBitQ alternative"** |
| 千万-million scale 单 vector TopK 推荐 | HNSW (动态) / NSG (静态) | **+ IVF + RaBitQ（new alternative）** |

### IVF + RaBitQ 作为 HNSW 替代的实证（NEW）

[per benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md Result 3 + Fig 4]

6 dataset 实测 IVF + RaBitQ + IVF rerank vs hnswlib HNSW：

| Dataset | RaBitQ + IVF | HNSW | 胜者 |
|---|---|---|---|
| Image | dominates | reference | **RaBitQ** |
| GIST | dominates | reference | **RaBitQ** |
| MSong | works (vs OPQ failure) | reference | **RaBitQ** |
| SIFT | comparable+ | reference | **RaBitQ ≈ HNSW** |
| DEEP | comparable+ | reference | **RaBitQ ≈ HNSW** |
| Word2Vec | dominates (OPQ unusable) | reference | **RaBitQ** |

→ **6/6 dataset RaBitQ + IVF dominate or equal HNSW**——这是 HNSW 自 2016 年发布以来的**最强工业 baseline 挑战**。但有 caveat：
- HNSW 用 hnswlib（M=16 default）；可能不是最优配置
- RaBitQ 论文为 RaBitQ favorable 的 baseline 选择
- 实测都在 ≤2.34M scale—— billion-scale HNSW 可能不同

### HNSW vs NSG 工程对比（不变）

[per concepts/hnsw.md + concepts/nsg.md]

| 工程考量 | HNSW | NSG |
|---|---|---|
| 构建成本 | 高 | **更低** |
| Million-scale 击败 | NSG 击败 | NSG 胜 |
| 增量更新 | ✓ | ✗ |
| Iterator + RM 集成 | ✓（VBASE 实证） | 理论可行未实证 |
| RaBitQ 替代 | n/a（graph 集成开放） | n/a（graph 集成开放） |

### 选择决策（updated）

- **简单 single-vector TopK + million scale + 静态数据 + 内存富余** → NSG
- **single-vector TopK + 增量数据 + million scale + 内存富余** → HNSW
- **single-vector TopK + 内存预算严** → **IVF + RaBitQ（NEW alternative）**
- **HCPS + memory + filter heavy** → HNSW + ACORN
- **复杂 multi-column / range / Join + SQL 用户** → HNSW + VBASE
- **billion-scale + memory** → HNSW + Faiss IVF coarse / **未来 RaBitQ + IVF 替代**
- **billion-scale + SSD** → DiskANN / SPANN（RaBitQ 集成开放）

### Open Questions

- **HNSW + RaBitQ neighbor list quantization**：理论上 graph node 邻居距离用 RaBitQ 估计（save memory）但 graph greedy traversal 与 batch SIMD 不 fit。NGT-QG 已在尝试类似思路但与 RaBitQ 不同。
- **NSG + RaBitQ**：同样开放
- **billion-scale RaBitQ vs HNSW** 实测：缺
- **RaBitQ + 增量数据**：P 矩阵 fixed 后增量数据用同 P；理论 OK 但 cluster 漂移如何处理？

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
