---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [concepts/product-quantization.md, systems/pase.md, benchmarks/pase-vs-cube-freddy.md]
cited-count: 3
---

# Post-snapshot (yang-2020-pase): quantization-landscape

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 用 IVFFlat（无 PQ）+ HNSW（无 PQ）——全程全精度**。这与 [SPANN](../../systems/spann.md) / [SPFresh](../../systems/spfresh.md) 同思路（不用 quantization）但驱动原因不同：SPANN/SPFresh 是 SSD-resident 路线下 quantization 没必要；**PASE 是 PG 8KB page-aligned 约束让 PQ 集成复杂**。论文显式选 IVFFlat 而非 IVFADC（PQ）—— "IVFFlat is straightforward and simple, with a faster index construction and less storage space requirements" + "100% recall when query vector is from candidate set"。

## Answer

### PASE quantization 选择（NEW）

[per systems/pase.md "ANN 算法 in PASE"]

PASE 论文 §2.2.1：

> "IVFFlat is a simplified version of IVFADC... When compared with other algorithms, IVFFlat is straightforward and simple, with a faster index construction and less storage space requirements. Moreover, a clustering method can be customized by users. The algorithm parameters are highly interpretable, and users can completely control the search accuracy by tuning these parameters."

→ PASE 主动选 **IVFFlat（无 PQ）** 而非 **IVFADC（PQ）**：
- 优势：simpler / faster build / 100% recall when query ∈ dataset / 用户可调
- 劣势：index size 大（GIST 1M IVFFlat 3912 MB vs Freddy IVFADC 116 MB——34×）

### Quantization landscape 完整版（updated with PASE）

| 方法 | 内存（128-d）| 是否 quantization | 集成系统 | 全/部分精度 |
|---|---|---|---|---|
| Binary | 16 byte | scalar quantization | Faiss | 部分 |
| SQ8 | 128 byte | scalar quantization | Faiss / Milvus / Manu | 部分 |
| PQ | 8-16 byte | product quantization | Faiss / Milvus / DiskANN / ADBV | 部分（ADC 失真） |
| OPQ | 同 PQ | rotated PQ | Faiss / Milvus（部分） | 部分 |
| RQ / LSQ | 类 PQ | additive quantization | Faiss / Manu | 部分 |
| ScaNN anisotropic | 同 PQ | score-aware PQ | ScaNN / Milvus | 部分 |
| **VGPQ** | 同 PQ | PQ + Voronoi 几何剪枝 | ADBV | 部分 |
| **IVFFlat（PASE）** | **全精度** | **无（cluster + 全精度 vector store）** | **PASE / pgvector** | **100% on dataset** |
| HNSW + 全精度 | 全精度 | 无 | Faiss / Milvus / PASE | 100% |
| SPANN posting list | 全精度 | 无 | SPANN / SPFresh | 100% |

→ PASE / SPANN / SPFresh 走相同"不用 quantization"路径——但驱动原因不同。

### "不用 quantization" 路线的不同动机（NEW）

[per systems/spann.md "不用 PQ —— 全程全精度", systems/pase.md, systems/spfresh.md]

| 系统 | 不用 PQ 的原因 |
|---|---|
| **SPANN** | SSD-resident posting list block-sequential read 带宽足 → 无需压缩 |
| **SPFresh** | LIRE 算法假设全精度（NPA check 在精确距离上）；融合 PQ 破坏收敛 |
| **PASE** | PG 8KB page constraint + IVFFlat 简单 + IVFADC 复杂集成 → 选简单 |

→ 三种"不用 quantization"原因不同——但都成功在各自场景下。**PASE 在 OLTP transactional 场景特别匹配**：transaction 频繁 + 增删改要原子，PQ codebook 重训成本不可接受；全精度 + PG WAL/transaction 一致。

### 与之前 ingest 的演进

| | wei-2020-analyticdb-v post | **yang-2020-pase post (NEW)** |
|---|---|---|
| Quantization 维度 | + VGPQ（partition geometry） | **+ PASE 走"不用 quantization"路径** |
| 不用 quantization 的系统数 | SPANN / SPFresh | **+ PASE** |
| 驱动"不用 PQ"的原因 | SSD storage / LIRE 算法 | **+ PG 8KB page + transaction-heavy** |

### Open / 未覆盖

- **PASE 集成 PQ 的工程难度**：论文未深入；但 IVFFlat → IVFADC 转换可能因 PG page layout 复杂
- **pgvector 是否集成 PQ**：wiki 未 ingest pgvector source
- **OPQ / RaBitQ 在 PG 内的可行性**：仍 zero coverage

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [systems/pase.md](../../systems/pase.md)
- [benchmarks/pase-vs-cube-freddy.md](../../benchmarks/pase-vs-cube-freddy.md)
