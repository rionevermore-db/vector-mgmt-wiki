---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [systems/spann.md, systems/diskann.md, systems/vbase.md, concepts/rabitq.md, concepts/product-quantization.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (gao-2024-rabitq): memory-vs-disk-large-scale

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 不直接给出新的 SSD 路线**——研究范畴是 in-memory ANN。但 RaBitQ 的 D-bit code + unbiased + sharp error bound 让现有 memory-vs-disk landscape 的 trade-off **整体右移**：(a) DiskANN 内存层 PQ 替换为 RaBitQ 后理论上更紧 + 更准（reduce SSD reads），(b) SPANN 反对量化的论点（"PQ 失真天花板"）被 RaBitQ 的 unbiased 性质削弱——SSD posting list 可能用 RaBitQ 编码节省 4× 占用。**核心 RaBitQ + SSD 集成实证 zero coverage**——这是 wiki 全新 frontier。

## Answer

### RaBitQ 在 memory-vs-disk landscape 的位置（NEW）

[per concepts/rabitq.md "工程实现要点"]

RaBitQ 论文 [gao-2024-rabitq §4-5] 全部实测 in-memory：
- 6 dataset 全 million-scale（最大 Image 2.34M）
- IVF + RaBitQ 全 RAM
- 不涉及 SSD / disk-resident 路径

→ 当前 RaBitQ 是**纯 memory 路径量化器**——但理论上可移植到 SSD-resident 系统。

### 路线对比表（updated with RaBitQ 潜在角色）

| 路线 | 数据驻留 | 量化 | RaBitQ 替代潜力 | scale 实证 |
|---|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | 不量化 | ✗ graph 集成开放 | 1B OOM |
| 磁盘 + 量化 + SSD re-rank（DiskANN）| DRAM (PQ) + SSD | **PQ** | **替换 PQ 后**：DRAM 占用 1/2 + 估计更准 → 减少 SSD reads | 1B SIFT |
| 磁盘 + IVF + SSD 全精度（SPANN）| DRAM (centroids) + SSD | 不量化 | **SSD posting list 用 RaBitQ**：占用 1/4 + 仍可 100% recall（rerank from SSD） | 1B+ Bing |
| 磁盘 + Filter-aware Vamana + PQ DRAM（Filtered-DiskANN）| DRAM (PQ + medoid) + SSD | PQ | 同 DiskANN | 28M DANN |
| 全内存 + Predicate-Agnostic HNSW（ACORN）| DRAM only | 不量化 | graph 集成开放 | 25M LAION |
| VBASE + HNSW (in-mem) | DRAM only | 不量化 | RaBitQ 集成 logical（IVF 走 RaBitQ）| 330K Recipe1M |
| VBASE + SPANN | DRAM (centroids) + SSD | 不量化 | 同 SPANN | 330K Recipe1M |
| **RaBitQ + IVF (RAM only)（NEW）** | **DRAM only** | **RaBitQ D bits** | n/a | 6 dataset, max 2.34M |
| **RaBitQ + DiskANN（理论可行）** | DRAM (RaBitQ) + SSD | RaBitQ | n/a | **完全空白** |
| **RaBitQ + SPANN（理论可行）** | DRAM + SSD (RaBitQ posting) | RaBitQ | n/a | **完全空白** |

→ 三条新 SSD 路径理论可行但 **zero coverage**。

### "RaBitQ + SSD" wiki 盲区（NEW）

[per systems/diskann.md Open Q + systems/spann.md Open Q]

| Workload | LCPS | HCPS | RaBitQ 集成 |
|---|---|---|---|
| memory-only | FilteredVamana / NHQ / ACORN / VBASE-HNSW | ACORN 唯一 | **RaBitQ + IVF（实证 6 dataset）** |
| memory + SSD | Filtered-DiskANN / VBASE+SPANN | **完全空白** | **完全空白** |
| Filter-aware build + RaBitQ | 完全空白 | 完全空白 | 完全空白 |

→ memory + SSD + RaBitQ 是 **wiki 全新最大盲区**——同时受 (a) RaBitQ graph 集成开放，(b) SPANN 反对量化的传统观点未挑战。

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| Memory-only filter-aware 选项 | + VBASE-HNSW iterator 330K | **+ RaBitQ + IVF 6 dataset** |
| Memory + SSD 选项 | Filtered-DiskANN 28M / VBASE+SPANN 330K | **不变**（RaBitQ 未集成 SSD 路径） |
| Quantizer 在 SSD 路径上的 trade-off | DiskANN PQ vs SPANN 全精度 | **+ RaBitQ 替代 PQ 的理论可行性** |
| HCPS + SSD | 完全空白 | **不变** |

### DiskANN PQ → RaBitQ 替换的理论分析（NEW）

[per concepts/rabitq.md + systems/diskann.md Open Q]

DiskANN 当前架构：
- DRAM: PQ codes (32 bytes/vector typical)
- SSD: full-precision vector + graph adjacency

RaBitQ 替换后：
- DRAM: RaBitQ D-bit code (D=128 → 16 bytes/vector)
- SSD: 不变

| 维度 | DiskANN PQ | DiskANN + RaBitQ |
|---|---|---|
| DRAM 占用（128-d） | ~32 bytes/vec | **16 bytes/vec (1/2)** |
| 距离估计精度 | biased | **unbiased + sharp bound** |
| Re-rank 触发 | beam-width 经验调 | **principled drop by bound** |
| Graph 集成 | mature | **research stage**（gao-2024 §4 future work） |

→ 理论上更紧更准但 graph 集成 challenge。

### 已知盲区（仍未覆盖）

- **DiskANN + RaBitQ 实证**：zero coverage
- **SPANN + RaBitQ 实证**：zero coverage（且 SPANN closure 与 RaBitQ normalization 兼容性理论开放）
- **HCPS + SSD + RaBitQ**：三重组合 zero coverage
- **Distributed RaBitQ**：单实例
- **GPU + RaBitQ**：bitwise + popcount 在 GPU 是否快于 PQ LUT？未实测

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/vbase.md](../../systems/vbase.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
