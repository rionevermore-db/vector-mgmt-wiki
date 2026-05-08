---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [concepts/acorn.md, systems/diskann.md, topics/disk-vs-memory-ann.md]
cited-count: 3
---

# Post-snapshot (patel-2024-acorn): memory-vs-disk-large-scale

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**ACORN 是 memory-only HNSW-based**——不直接覆盖 SSD 路径。LAION 25M（59 GB index）在 m5d.24xlarge 370 GB RAM 内存可行；更大 scale 需要 SSD 路径但 ACORN 论文未实测 ACORN+SSD 组合。**memory-vs-disk landscape 中 ACORN 仅 memory 路径**——HCPS filter 场景的 SSD 解仍是开放（FilteredVamana + DiskANN 走 LCPS SSD；ACORN+SSD 待研究）。

## Answer

### ACORN 在 disk-vs-memory landscape 的位置（NEW）

[per concepts/acorn.md "Open Questions"]

ACORN 论文实测全部 in-memory：
- LAION 25M, 59 GB index, m5d.24xlarge 370 GB RAM
- ACORN-γ TTI 38007 s（10.5 小时 build）
- ACORN-γ 比 HNSW 1.3× index size

**SSD 路径 ACORN 论文未涉及**——HNSW SSD 集成本身困难（neighbor random access）继承到 ACORN。

### 路线对比表（ACORN 仅 memory 路径，[Fig table 不更新])

| 路线 | 数据驻留 | filter capability | scale 实证 |
|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | ✗ | 1B OOM |
| 磁盘 + 量化导航 + SSD re-rank（DiskANN）| DRAM (PQ) + SSD | ✗ | 1B SIFT |
| 磁盘 + IVF + SSD 全精度（SPANN）| DRAM (centroids) + SSD | ✗ | 1B+ Bing |
| 磁盘 + Filter-aware Vamana + PQ DRAM（Filtered-DiskANN）| DRAM (PQ + medoid map) + SSD | ✓ build-time, LCPS | **28M DANN** |
| **全内存 + Predicate-Agnostic HNSW**（ACORN，NEW）| **DRAM only** | **✓ build-time, HCPS unbounded** | **25M LAION** |

→ ACORN 是 wiki 内 **HCPS-supporting** 唯一方案，但仅 memory 路径——大 scale + HCPS 路径仍是开放。

### "HCPS + SSD" 的 wiki 盲区（NEW）

| Workload | LCPS（≤1000 filters）| HCPS（10^8+ filters） |
|---|---|---|
| memory-only | FilteredVamana / NHQ / **ACORN** | **ACORN 唯一** |
| memory + SSD | **Filtered-DiskANN** ✓ | **完全空白** |

→ HCPS + 1B+ scale + SSD 路径是 wiki 内**最大盲区**——ACORN+SSD 论文 §8 提"future work"。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| Filter-aware SSD 实证 | 28M DANN (Filtered-DiskANN) | **不变** |
| Memory-only filter-aware 选项 | FilteredVamana 28M DANN（也支持 SSD） | **+ ACORN 25M LAION HCPS** |
| HCPS + SSD 路径 | 未触及 | **明确：完全空白** |

### 已知盲区

- **ACORN + SSD storage**：HNSW SSD 困难继承
- **HCPS + 1B+ scale**：完全空白
- **ACORN 多节点分布式**：单机
- **CXL / 持久内存 + ACORN**：仍未覆盖

## Cited Pages

- [concepts/acorn.md](../../concepts/acorn.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
