---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [concepts/product-quantization.md, concepts/scann.md, systems/faiss.md, systems/milvus.md, benchmarks/pq-sift-recall.md, benchmarks/scann-glove1.2m-mips.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, topics/index-selection.md]
cited-count: 8
---

# Post-snapshot: quantization-landscape

## TL;DR

Wiki 仍**只有 PQ + ScaNN 完整 page**，OPQ/RQ/LSQ/PRQ/PLSQ 仅在 Faiss 量化家族层级中提及。**RaBitQ 仍未覆盖**。**新增 Milvus 视角的工业组合证据**：Milvus 同时实现 IVF_FLAT / IVF_SQ8 / IVF_PQ + HNSW + RNSG，**不实现 OPQ**（论文显式说 "We exclude LSH-based approaches"，但 OPQ 这种 PQ 变体也未在论文 §2.2 主索引列表里）；**自研 SQ8H** 为 hybrid CPU/GPU 索引（IVF_SQ8 + 步骤 1 GPU + 步骤 2 CPU），是 PQ 框架外的工程创新。

## Answer

### 量化家族层级 [per concepts/product-quantization.md "Quantizer 家族中的位置"]

```
binary (1-bit scalar)
  ⊂ scalar quantizer (per-dim, 4/6/8 bit)
    ⊂ product quantizer (PQ)
      ⊂ product-additive quantizer (PRQ, PLSQ)
        ⊂ additive quantizer (RQ, LSQ)
          ⊂ general MCQ
```

### 各方法精度-速度-内存（updated with Milvus）

| 方法 | 内存（128-d） | 训练成本 | 距离精度 | wiki 覆盖 | Milvus 集成 |
|---|---|---|---|---|---|
| Binary (1-bit scalar) | 16 byte | 低 | 低 | 仅 hierarchy | ✗ |
| Scalar Quantizer (SQ8) | 128 byte | 极低 | 中 | hierarchy + faiss | ✓ **IVF_SQ8** [wang-2021-milvus §2.2] |
| **SQ8H（NEW，Milvus 自研）** | 同 SQ8 | 同 SQ8 | 同 SQ8 | **systems/milvus.md** | ✓ **GPU step 1 + CPU step 2 hybrid**，PCIe 瓶颈下系统性优于 pure CPU/GPU [per benchmarks/milvus-vs-prior-sift10m-deep10m.md Fig 13] |
| Scalar Quantizer (SQ6/4) | 96/64 byte | 极低 | 中-低 | hierarchy + faiss-trillion | ✗（Milvus 论文未列） |
| **Product Quantizer (PQ)** | **8-16 byte** typical | 中 | 中-低 | **完整 page** | ✓ **IVF_PQ** |
| Optimized PQ (OPQ) | 同 PQ | 中-高 | 比 PQ 高 | 仅提及 | ✗（Milvus 论文未列；后续 Knowhere 库可能集成但未进 SIGMOD 论文） |
| Residual Quantizer (RQ) | 类 PQ | 高 | 比 PQ 高（小 code） | 提及 | ✗ |
| Local Search Quantizer (LSQ) | 类 PQ | 高 | 比 PQ / RQ 都高 | 提及 | ✗ |
| ScaNN (Anisotropic PQ) | 同 PQ | 类 PQ | **MIPS 任务 +8pp Recall1@10** | **完整 page** | ✗ |
| RaBitQ | — | — | — | **wiki 未覆盖** | — |

### 与 IVF / HNSW 的工业组合（**新增 Milvus 实测数据**）

| 组合 | 用途 | 工业代表 | 量化 |
|---|---|---|---|
| Flat | 全表 brute force | 千万级 baseline | 无 |
| **IVF_FLAT** | IVF 倒排桶 + 不压缩 | Milvus 默认（SIFT1B 单节点 1.5 TB RAM 全装内存） | 无 |
| **IVF_SQ8** | IVF + 标量量化 | Milvus IVF_SQ8 [wang-2021-milvus] | SQ8 |
| **IVF_SQ8H** | **IVF_SQ8 + GPU/CPU hybrid** | **Milvus 自研**，SIFT1B 装不进 GPU 时最优 | SQ8 |
| IVFPQ | IVF + PQ | Faiss IndexIVFPQ；Milvus IVF_PQ | PQ |
| IVF + SQ6 + HNSW coarse | trillion-scale 路线 | Meta 1.5T [per benchmarks/faiss-trillion-scale.md] | SQ6 |
| HNSW (full) | 不量化、全图 | million-scale 标准 | 无 |
| HNSW-as-coarse-quantizer + PQ | 大 IVF list 数下加速 coarse | Faiss `IVF65536_HNSW32,PQ32` | PQ |
| Scalar 8-bit re-rank | IVFPQ 后用 SQ8 精化 top-k | Faiss `IndexRefine` | SQ8 re-rank |
| ScaNN 4-bit FastScan | SIMD 加速 ADC | ScaNN；Faiss FastScan 反向借鉴 | 4-bit |
| **Milvus segment-level mix** | **每 segment 可独立选 index** | **Milvus** [per systems/milvus.md] | 任意（segment 级粒度） |

### Milvus SQ8H 的工程意义 [per systems/milvus.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md]

**问题**：Faiss IVF_SQ8 在 SIFT1B（数据装不进 GPU 16 GB）时按需 PCIe 传 bucket → 利用率 1-2 GB/s（vs 15.75 GB/s 上限），且 small batch 下 GPU 反而比 CPU 慢。

**Milvus 解**（Algorithm 1）：
- 大 batch（≥1000）+ centroids 装得进 GPU memory：step 1 GPU 找 n_probe centroids；step 2 CPU 扫 bucket（避免传 large data）
- 小 batch：fallback 纯 CPU

实测 [Fig 13]：SQ8H 系统性优于 pure CPU SQ8 与 pure GPU SQ8，差距随 query batch size 增长扩大。

> **wiki 解读**：SQ8H 不是新的 quantizer，而是**已有 quantizer (SQ8) 的 hybrid 调度**。这是 wiki 内继 [WarpSelect](../../concepts/warpselect.md) "全 GPU 寄存器" 路线之后的另一条 GPU 优化路径——hybrid scheduling 而非 fused kernel。

### Open / 未覆盖

- **OPQ 独立 page**：旋转 + 维度重排，wiki 仅在 PQ page 一处提及
- **Residual / Local Search / Product LSQ**：Faiss 家族表里有，无 detail
- **RaBitQ**：完全未覆盖
- **量化 + GPU 路线**：[per benchmarks/faiss-gpu-sift1b-deep1b.md] IVFPQ on GPU；OPQ/RQ on GPU 工程未覆盖
- **DiskANN 的 DRAM-PQ + SSD-FullPrecision 混合模式**：[per concepts/product-quantization.md §DRAM-PQ + SSD-FullPrecision]
- **SPANN 反例**：[per concepts/product-quantization.md §反例] IVF 不必绑 PQ
- **Milvus 2.0+ 量化策略**：本 ingest 是 1.x 论文；2.0+ cloud-native 重写后 quantizer 选择可能不同

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/scann.md](../../concepts/scann.md)
- [systems/faiss.md](../../systems/faiss.md)
- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/pq-sift-recall.md](../../benchmarks/pq-sift-recall.md)
- [benchmarks/scann-glove1.2m-mips.md](../../benchmarks/scann-glove1.2m-mips.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [topics/index-selection.md](../../topics/index-selection.md)
