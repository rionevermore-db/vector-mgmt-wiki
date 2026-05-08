---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [concepts/product-quantization.md, concepts/acorn.md]
cited-count: 2
---

# Post-snapshot (patel-2024-acorn): quantization-landscape

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**ACORN 不引入新 quantizer**——继承 HNSW 全精度 base + predicate-agnostic 改造。但 ACORN 论文 §6.1 的 **predicate-agnostic compression**（保留前 M_β candidates uncompressed，剩余 truncate）是新型"邻居列表压缩"思路——与 PQ / VGPQ 等"向量压缩"思路正交。这是 wiki 内**首次见 graph index 的 neighbor-list 压缩**作为 quantization landscape 维度。

## Answer

### ACORN 的"邻居列表压缩"（NEW）

[per concepts/acorn.md "Predicate-Agnostic Pruning"]

ACORN-γ 构造时每 node 收 M·γ candidate edges。直接存所有候选导致 index size 爆。**Predicate-agnostic compression** 策略：

```
保留前 M_β candidates uncompressed（典型 M_β = 32 / 64 / 128）
剩余 M·γ - M_β candidates truncate（保 search-time 2-hop 扩展恢复）
```

**与传统 quantization 的差异**：
- PQ / OPQ / SQ / VGPQ：**压缩 vector 数据**（节省存储 + 加速距离计算）
- ACORN compression：**截断邻居列表**（节省 graph storage + 维护 search 性能）

→ wiki 内首次见 graph index 的 neighbor-list 压缩作为 quantization 类技术。

### Quantization landscape 完整版（updated with ACORN）

| 方法 | 类型 | 内存 / size 节省 | 集成系统 | 维度 |
|---|---|---|---|---|
| Binary | scalar quantization | 32× | Faiss | 向量 |
| SQ8 | scalar quantization | 4× | Faiss / Milvus | 向量 |
| PQ | product quantization | 16-32× | Faiss / Milvus / DiskANN | 向量 |
| OPQ | rotated PQ | 16-32× | Faiss | 向量 |
| RQ / LSQ | additive quantization | 16-32× | Faiss / Manu | 向量 |
| ScaNN anisotropic | score-aware PQ | 16-32× | ScaNN / Milvus | 向量 + loss |
| VGPQ | PQ + Voronoi 几何剪枝 | 16-32× + scan reduction | AnalyticDB-V | 向量 + partition |
| **ACORN compression** | **neighbor list truncation** | **graph size reduction** | **ACORN-γ** | **graph edges** |

→ ACORN compression 是**正交的新维度**——可与所有 vector quantization 叠加。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| Quantization 维度 | vector quantization | **+ graph neighbor list compression** |
| 与 filter-aware 的耦合 | DiskANN PQ + filter（FilteredVamana 隐含）| **+ ACORN compression + filter-agnostic build** |
| Pruning 哲学 | metadata-aware RNG（FilteredVamana）| **predicate-agnostic compression** |

### Open / 未覆盖

- **ACORN compression + PQ vector compression 叠加**：理论可行；ACORN 论文未试
- **ACORN compression 在其他 graph base（NSG / Vamana）的可行性**：未试
- **RaBitQ**：仍未覆盖
- **ACORN + SSD storage**：HNSW SSD 集成困难继承到 ACORN

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/acorn.md](../../concepts/acorn.md)
