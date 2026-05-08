---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [concepts/hnsw.md, concepts/acorn.md, concepts/filtered-vamana.md, benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
cited-count: 4
---

# Post-snapshot (patel-2024-acorn): hnsw-vs-nsg-selection

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**HNSW 在 filter-heavy + unbounded predicate 场景的地位逆袭**：之前 FilteredVamana 让 Vamana 在 filter-aware 场景超越 HNSW；ACORN 让 **HNSW 通过 predicate-agnostic 改造重夺优势**——比 FilteredVamana 在所有 LCPS dataset 快 2-10×。**HNSW vs NSG 选型轴**新增 "filter set cardinality + operator"——HCPS workload 下 ACORN-modified HNSW 是首选；NSG 在所有 filter-heavy 场景仍未涉及。

## Answer

### Graph ANNS 选型完整轴（updated）

[per concepts/hnsw.md "ACORN" + concepts/filtered-vamana.md "与 ACORN 的对比"]

| 场景 | 推荐 | 备注 |
|---|---|---|
| 静态 + 高 recall（无 filter） | NSG（论文实测最强）| 不增量 |
| 增量 add（无 filter） | **HNSW** | 生态最广 |
| Filter-heavy + LCPS（≤1000 equality） | **FilteredVamana / StitchedVamana**（Vamana base） | Microsoft 广告 +35% revenue 实证 |
| **Filter-heavy + HCPS（10^8+）+ 任意 operator** | **ACORN-γ / ACORN-1**（HNSW base） | Stanford 实证；25M LAION >1000× over baselines |
| Filter-heavy + 实时 streaming insert | ACORN-1 | construction 同 HNSW |
| SSD-resident + filter | Filtered-DiskANN（Vamana + SSD） | 28M DANN 实证 |
| SSD-resident 无 filter | Vamana / DiskANN | 1B SIFT 实证 |

→ **HNSW 路径**因 ACORN 重新成为 filter-heavy 主选——NSG 仍无 filter-aware 变体。

### HNSW 在 ACORN 中的定位（NEW）

[per concepts/hnsw.md "ACORN"]

ACORN 是 HNSW 的 **minor extension**：
- 保持 HNSW multilayer hierarchy + level assignment
- Construction 改：每 node 收 M·γ candidate（vs M）
- Pruning 改：predicate-agnostic compression（vs RNG-based）
- Search 改：GET-NEIGHBORS 加 predicate filter（ACORN-γ filter+truncate；ACORN-1 + 2-hop）

→ ACORN 可在**现有 HNSW 库**实现（如 hnswlib / Faiss IndexHNSW）——这是 HNSW 生态优势。NSG 没有同等 mature 库，因此 NSG-based predicate-agnostic 论文未出现。

### LCPS 上 ACORN 反超 FilteredVamana（NEW，关键观察）

[per benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md "主结果 1"]

LCPS（FilteredVamana 主场，cardinality 12 random label）：
- **ACORN-γ 比 FilteredVamana 2-10× higher QPS @ 0.9 recall**
- 即使 FilteredVamana 是 specialized index，也被 generic ACORN 反超

**为什么 HNSW base 反超 Vamana base?**
- HNSW multilayer hierarchy 给 ACORN 更好的 navigability
- Vamana single-layer + α-controlled pruning 在 predicate subgraph 上稀疏化困难
- ACORN-γ 收 M·γ candidates 的 dense graph 让 predicate subgraph 更连通

→ **HNSW 不只在无 filter 场景生态广，在 filter-heavy 场景也回归领先**。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| HNSW 在 filter 场景 | post-processing 在 1pc 失败 | **ACORN 改造后 LCPS 反超 FilteredVamana** |
| Filter-aware build base | Vamana 唯一（FilteredVamana） | **HNSW（ACORN）+ Vamana（FilteredVamana）双轨** |
| NSG 在 filter 场景 | 仍未涉及 | 仍未涉及 |
| HNSW 生态优势 | 隐含 | **显式：ACORN 可在 hnswlib / Faiss 实现** |

### Open / 未覆盖

- **NSG + ACORN-style 改造**：理论可行未尝试；NSG 单层 + medoid 单 entry point 与 ACORN multilayer 假设冲突
- **HNSW + FilteredVamana-style label baked-in build**：理论可行未尝试
- **Vamana + ACORN-style predicate-agnostic**：理论可行；预测可结合 Vamana α-controlled pruning + predicate-agnostic compression

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/acorn.md](../../concepts/acorn.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)
