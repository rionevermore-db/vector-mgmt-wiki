---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [concepts/product-quantization.md, concepts/relaxed-monotonicity.md]
cited-count: 2
---

# Post-snapshot (zhang-2023-vbase): quantization-landscape

## TL;DR (delta from patel-2024-acorn post)

**VBASE 不引入新 quantizer**——研究范畴是 query engine 范式（iterator + RM），与 quantization 维度正交。**唯一新增**：VBASE 实测 IVFFlat（全精度，无 PQ）作为 RM iterator 集成对象——与 SPANN 的"全精度 posting list" 哲学一致；论文未实测 IVFPQ。这暗示 **lossy quantization 在 RM 框架下的精度损失**仍是未触及的开放问题——iterator 的 Phase 2 检测可能因 PQ 距离误差而误判。

## Answer

### Quantization landscape（不变）

| 方法 | 类型 | 内存 / size 节省 | 集成系统 | 维度 |
|---|---|---|---|---|
| Binary | scalar quantization | 32× | Faiss | 向量 |
| SQ8 | scalar quantization | 4× | Faiss / Milvus | 向量 |
| PQ | product quantization | 16-32× | Faiss / Milvus / DiskANN | 向量 |
| OPQ | rotated PQ | 16-32× | Faiss | 向量 |
| RQ / LSQ | additive quantization | 16-32× | Faiss / Manu | 向量 |
| ScaNN anisotropic | score-aware PQ | 16-32× | ScaNN / Milvus | 向量 + loss |
| VGPQ | PQ + Voronoi 几何剪枝 | 16-32× + scan reduction | AnalyticDB-V | 向量 + partition |
| ACORN compression | neighbor list truncation | graph size reduction | ACORN-γ | graph edges |

→ VBASE 不增加新行；它**正交于** quantization。

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| 新增 quantizer | ACORN compression（graph 维度） | **无** |
| Quantization 与 query interface 关系 | 同前 | **新触及：lossy quantization 是否影响 RM Phase 检测？** |
| VBASE 实测 quantizer | n/a | **仅 IVFFlat（全精度）**——IVFPQ / VGPQ 未实测 |

### Lossy Quantization 在 RM 框架下的开放问题（NEW）

[per concepts/relaxed-monotonicity.md "Open Questions"]

VBASE iterator 的 RM Phase 2 检测依赖**真实距离**：

```c
window_median = median(D(q, x_{t-W+1}), ..., D(q, x_t));  // 移动中位数
if window_median >= R_q:
    enter Phase 2  // 自动停
```

但 PQ-based 索引（IVFPQ / VGPQ）返回的是 **lossy distance estimation**：

```c
D_pq(q, x_i) = approximate_distance ± quantization_error
```

→ 三种潜在问题：
1. **False Phase 2**：PQ 误差让 D_pq 提前 ≥ R_q，过早 stop（recall 损失）
2. **False not-Phase 2**：PQ 误差让 D_pq 仍 < R_q，过晚 stop（latency 损失）
3. **Result equivalence 破坏**：[zhang-2023-vbase §4.4] 的等价证明依赖 exact distance；PQ lossy 时证明不再 hold

**VBASE 论文 §4 未涉及此问题**——只实测 IVFFlat / HNSW（全精度）+ SPANN（全精度 posting list）。lossy quantization 与 iterator 范式的兼容性是 wiki 内**全新的**开放问题。

### 已知盲区

- **IVFPQ / VGPQ + RM**：精度损失影响未实测
- **DiskANN PQ + RM**：DiskANN 内存层用 PQ 压缩 + SSD re-rank；re-rank 后的 RM 行为？理论上 re-rank 后 distance accurate 再做 RM 检测可行，但 VBASE 未集成 DiskANN
- **ACORN compression + IVFPQ vector compression 叠加**：理论可行；ACORN 论文未试，VBASE 未涉及
- **RaBitQ**：仍未覆盖

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/relaxed-monotonicity.md](../../concepts/relaxed-monotonicity.md)
