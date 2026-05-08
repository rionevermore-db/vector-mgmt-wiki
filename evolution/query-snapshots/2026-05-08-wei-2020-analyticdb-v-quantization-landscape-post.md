---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [concepts/product-quantization.md, concepts/vgpq.md, concepts/scann.md, systems/analyticdb-v.md, benchmarks/analyticdb-v-vs-twostep.md]
cited-count: 5
---

# Post-snapshot (wei-2020-analyticdb-v): quantization-landscape

## TL;DR (delta from guo-2022-manu post)

**新增 IVFPQ successor [VGPQ](../../concepts/vgpq.md)**——Voronoi diagram + neighbor midpoints 切 subcells 的几何剪枝。同 IVFPQ index size，构造时间 -10%，**recall vs response time 在 SIFT1B / Deep1B / AliCommodity 三 dataset 全程优于 IVFPQ**。VGPQ 与 [ScaNN anisotropic loss](../../concepts/scann.md) 是**正交的 IVFPQ 改进方向**——分别从 partition geometry vs quantization loss 入手。

## Answer

### VGPQ 在 quantization landscape 的位置（NEW）

[per concepts/vgpq.md "与 PQ / IVFPQ 的对比"]

```
PQ (2011) → IVFPQ (2011, IVF + PQ 加倒排剪枝)
              ├─→ ScaNN (2019) 改 quantization loss（anisotropic）
              ├─→ VGPQ (2020) 改 partition geometry（Voronoi subcell）  ← NEW
              └─→ OPQ / RQ / LSQ / PRQ / PLSQ 量化变体
```

VGPQ 与 ScaNN 都是 IVFPQ successor 但思路不同：

| | ScaNN | **VGPQ** |
|---|---|---|
| 改进维度 | quantization loss | partition geometry |
| 优化目标 | MIPS recall | recall-vs-latency on L2 / IP / Hamming |
| 集成系统 | Google ScaNN / Faiss FastScan | AnalyticDB-V batching layer |
| 是否依赖特定任务 | MIPS 优势最大 | 全任务通用 |
| 索引大小 | 同 PQ | **同 PQ** |
| 同 IVFPQ 叠加 | ✓（替换 loss） | ✓（替换 partition） |
| 两者叠加 | 理论可行未试 | 同上 |

→ wiki 已 ingest 三个 IVFPQ successor 算法（OPQ in 学术家族、ScaNN、VGPQ），所有都同 index size 但不同思路。

### Quantization landscape 完整版（updated）

| 方法 | 内存（128-d） | 学习成本 | 精度增益 vs IVFPQ | 集成系统 | 特点 |
|---|---|---|---|---|---|
| Binary | 16 byte | 极低 | 低 | Faiss | 1-bit |
| SQ8 | 128 byte | 极低 | 低 | Faiss / Milvus | 8-bit per dim |
| SQ6 | 96 byte | 极低 | 低 | Faiss trillion-scale | 6-bit per dim |
| PQ | 8-16 byte | 中 | baseline | Faiss / Milvus / DiskANN | 子向量 k-means |
| OPQ | 同 PQ | 中-高 | +10-20% | Faiss / Milvus（部分） | 旋转预处理 |
| RQ | 类 PQ | 高 | +15-25% (small code) | Faiss / Milvus | 顺序 residual quantize |
| LSQ | 类 PQ | 高 | +20-30% | Faiss | simulated annealing |
| **ScaNN (anisotropic PQ)** | 同 PQ | 类 PQ + 加权伪逆 | **+8pp Recall1@10 (MIPS)** | ScaNN / Milvus | score-aware loss |
| **VGPQ**（NEW） | **同 PQ** | **类 PQ + Voronoi 构造** | **全程 < IVFPQ response time** | **AnalyticDB-V** | **partition geometry** |
| RaBitQ | — | — | — | wiki 未覆盖 | — |

### ADBV 不用 OPQ / RQ / LSQ 的原因（NEW，论文未明示但可推断）

ADBV 与 [Manu](../../systems/milvus.md) 不同——Manu Table 1 列 OPQ/RQ/SQ 全套，ADBV 仅 PQ / VGPQ + SQ。可能解释：
1. ADBV 自家 VGPQ 替代了 OPQ / RQ / LSQ 的 partition 几何角色——继续这些 quantizer 收益边际
2. ADBV 主打 hybrid query（attribute filter + vector）——quantizer 优化对 hybrid 影响小于对纯 vector
3. ADBV production 已稳定 → quantizer 试错成本高

### 与之前 ingest 的演进

| | guo-2022-manu post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| IVFPQ successor 数 | ScaNN | **+ VGPQ** |
| Quantizer 创新维度 | quantization loss / GPU layout | **+ partition geometry** |
| 与 IVFPQ 同 index size 的方法 | ScaNN | **+ VGPQ** |
| 工业实测大规模 quantizer | Faiss SQ6 trillion / DiskANN PQ | **+ ADBV VGPQ 13B records production** |

### Open / 未覆盖

- **VGPQ 在 Faiss / Milvus 的 portability**：算法公开，但其他系统未集成；可能因高维 Voronoi 几何不稳定
- **VGPQ + ScaNN anisotropic 叠加效果**：未试
- **RaBitQ**：仍未覆盖

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [concepts/scann.md](../../concepts/scann.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [benchmarks/analyticdb-v-vs-twostep.md](../../benchmarks/analyticdb-v-vs-twostep.md)
