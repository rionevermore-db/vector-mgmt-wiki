---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [concepts/product-quantization.md, concepts/scann.md, systems/faiss.md, benchmarks/pq-sift-recall.md, benchmarks/scann-glove1.2m-mips.md, topics/index-selection.md]
cited-count: 6
---

# Pre-snapshot: quantization-landscape

## TL;DR

Wiki 当前**只有 PQ / IVFADC（核心）+ ScaNN（anisotropic 变体）+ Faiss 量化家族层级**。OPQ、Scalar Quantizer、Residual Quantizer、Local Search Quantizer 仅在 [systems/faiss.md] 与 [concepts/product-quantization.md] 的家族层级表中提及，无独立 page。**RaBitQ 完全未覆盖**。组合实践：IVFPQ 是 Faiss 千亿级标准；HNSW + PQ 主要做 coarse quantizer；ScaNN 4-bit interleaved SIMD 被 Faiss FastScan 反向借鉴。

## Answer

### 量化家族层级（[per concepts/product-quantization.md "Quantizer 家族中的位置"]）

```
binary (1-bit scalar)
  ⊂ scalar quantizer (per-dim, 4/6/8 bit)
    ⊂ product quantizer (PQ)
      ⊂ product-additive quantizer (PRQ, PLSQ)
        ⊂ additive quantizer (RQ, LSQ)
          ⊂ general MCQ
```

每层比上一层有更多自由度（更准）但也更贵（更慢、训练更多）。

### 各方法精度-速度-内存

| 方法 | 内存（128-d 向量） | 训练成本 | 距离精度 | wiki 覆盖 |
|---|---|---|---|---|
| Binary (1-bit scalar) | 16 byte | 低 | 低 | 仅 hierarchy 提及 |
| Scalar Quantizer (SQ8) | 128 byte | 极低 | 中（与原向量同构） | 仅在 Faiss 家族中提及；trillion-scale 案例用 SQ6 [per benchmarks/faiss-trillion-scale.md] |
| Scalar Quantizer (SQ6/4) | 96/64 byte | 极低 | 中-低 | 同上 |
| **Product Quantizer (PQ)** | **8-16 byte** typical | 中（per-segment k-means） | 中-低（reconstruction loss） | **完整 page**：[concepts/product-quantization.md] |
| Optimized PQ (OPQ) | 同 PQ | 中-高（旋转优化） | 比 PQ 高（去相关） | 仅提及；无独立 page |
| Residual Quantizer (RQ) | 类 PQ | 高（顺序量化） | 比 PQ 高（小 code 优势）[per concepts/product-quantization.md §Quantizer 家族] | 提及无 page |
| Local Search Quantizer (LSQ) | 类 PQ | 高（simulated annealing） | 比 PQ / RQ 都高 | 提及无 page |
| ScaNN (Anisotropic PQ) | 同 PQ | 类 PQ + 加权伪逆 | **MIPS 任务下 +8pp Recall1@10**（Glove1.2M 200 bit 0.83→0.91）[per benchmarks/scann-glove1.2m-mips.md] | **完整 page**：[concepts/scann.md] |
| RaBitQ | — | — | — | **wiki 未覆盖** |

经验区间 [per concepts/product-quantization.md §Quantizer 家族]：
- **小 code (<32 byte)**：LSQ / RQ 优于 PQ
- **大 code (>64 byte)**：PRQ / PLSQ 接管
- **MIPS 任务**：ScaNN 的 anisotropic 与所有方法**正交**，可叠加

### 与 IVF / HNSW 的组合

[per systems/faiss.md, topics/index-selection.md]：

| 组合 | 用途 | 工业代表 |
|---|---|---|
| Flat PQ | 全表 ADC 扫描 | 千万级单机 |
| **IVFPQ (= IVF + PQ)** | **千亿级标准**：coarse 倒排 + PQ 压缩 ADC | Faiss `IndexIVFPQ`、PQ 论文原型 [per concepts/product-quantization.md] |
| IVFPQ + OPQ | 加旋转预处理 | Faiss `OPQ16_64,IVF1024,PQ16` factory string |
| IVF + SQ6 + HNSW coarse | trillion-scale 路线 | Meta 1.5T [per benchmarks/faiss-trillion-scale.md] |
| HNSW (full) | 不量化、全图 | million-scale 标准 |
| HNSW-as-coarse-quantizer + PQ | 大 IVF list 数下加速 coarse 查询 | Faiss `IVF65536_HNSW32,PQ32` [per topics/index-selection.md Step 2] |
| Scalar 8-bit re-rank | 跑完 IVFPQ 后用 SQ8 精化 top-k | Faiss `IndexRefine` |
| ScaNN 4-bit FastScan | SIMD 加速 ADC | ScaNN 库；Faiss FastScan 反向借鉴 [per concepts/scann.md] |

### Open / 未覆盖

- **OPQ 独立 page**：旋转 + 维度重排，wiki 仅在 PQ page 一处提及
- **Residual / Local Search / Product LSQ**：Faiss 家族表里有，无 detail
- **RaBitQ**：完全未覆盖
- **量化 + GPU 路线**：[per benchmarks/faiss-gpu-sift1b-deep1b.md] 只覆盖 IVFPQ on GPU；OPQ/RQ on GPU 工程未覆盖
- **DiskANN 的 DRAM-PQ + SSD-FullPrecision 混合模式**：[per concepts/product-quantization.md §DRAM-PQ + SSD-FullPrecision] 这是 wiki 现有最深入的一例混合
- **SPANN 反例**：[per concepts/product-quantization.md §反例] IVF 不必绑 PQ；SSD 全精度也能 1B 单机

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/scann.md](../../concepts/scann.md)
- [systems/faiss.md](../../systems/faiss.md)
- [benchmarks/pq-sift-recall.md](../../benchmarks/pq-sift-recall.md)
- [benchmarks/scann-glove1.2m-mips.md](../../benchmarks/scann-glove1.2m-mips.md)
- [topics/index-selection.md](../../topics/index-selection.md)
