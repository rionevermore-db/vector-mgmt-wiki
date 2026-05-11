---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, concepts/rabitq.md, concepts/product-quantization.md]
cited-count: 3
---

# Post-snapshot (weaviate-docs): quantization-landscape

## TL;DR (delta from qdrant-docs post)

**Weaviate 引入 RQ (Rotational Quantization) 作为 default**——与 [RaBitQ](../../concepts/rabitq.md) **同源**（random rotation + quantize）但选 **8-bit per dim** 而非 1-bit (RaBitQ)。这暴露**"rotation-quantization 范式"在 production 上多粒度选择**：RaBitQ 1-bit (32× compression) 极致 / Weaviate RQ8 (4× compression) 平衡 / Weaviate RQ1 (32× compression) 与 RaBitQ 同粒度但 Weaviate 不声明理论 bound。**production OSS vector DBMS quantization landscape**：Milvus PQ/SQ8/ScaNN / Qdrant Scalar/Binary/1.5-2bit/Asymmetric/PQ / **Weaviate RQ8/RQ1 default**——三家都不集成严格 RaBitQ 论文版本（unbiased + sharp bound），但都在 binary/rotation path 有 production 实现。

## Answer

### Weaviate RQ vs RaBitQ 关系（NEW）

[per sources/docs/weaviate/llms.txt §Architecture + Best Practices + concepts/rabitq.md]

| 维度 | [RaBitQ](../../concepts/rabitq.md) | **Weaviate RQ8** | **Weaviate RQ1** |
|---|---|---|---|
| 核心思路 | random orthogonal rotation + quantize | **同**（random rotation + quantize） | **同** |
| Per-dim bits | 1 (binary) | 8 | 1 |
| Compression | 32× | 4× | 32× |
| Theoretical guarantee | **O(1/√D) sharp w.h.p.** | 经验调参 (good recall/speed tradeoff) | 经验调参 (cost-sensitive workloads) |
| Production deployment | Stanford 学术 only (until production)| **Weaviate default since 1.x** | Weaviate optional |
| Estimator | unbiased | (likely biased, docs 未声明) | (likely biased) |
| Error bound rerank | drop-by-bound | 无 explicit error bound | 无 explicit error bound |
| Wiki 内地位 | concept | systems-only impl detail | systems-only impl detail |

→ **RaBitQ 是"理论保证的 rotation-quantization"**；**Weaviate RQ 是"production 工程化的 rotation-quantization"**——同源不同 design priority。**Weaviate 不声明 RQ 与 RaBitQ 关系**（docs 未提）但技术上是 sibling。

### Quantization landscape（updated with Weaviate RQ）

| 方法 | 类型 | error bound | 主要 use case | 实证 |
|---|---|---|---|---|
| Binary / SQ8 | scalar | 无 | 内存极限 | Faiss / Qdrant SQ default |
| PQ / OPQ / LSQ | product | 无 | 减内存 | Faiss / Milvus / DiskANN |
| ScaNN anisotropic | score-aware | 无 | MIPS | Google ScaNN / Milvus SCANN |
| VGPQ | PQ + Voronoi | 无 | OLAP | ADBV |
| ACORN compression | graph edges | 无 | predicate-agnostic | ACORN |
| **RaBitQ** | **hypercube + random rotation 1-bit** | **O(1/√D) sharp w.h.p.** | 理论保证 | **Stanford 学术 + 跨 production gap** |
| Qdrant Scalar (SQ) | scalar | 无 | Qdrant default | Qdrant production |
| Qdrant Binary 1-bit | binary | 无 + rescore needed | 32× + 40× speedup | Qdrant production |
| Qdrant 1.5/2-bit | binary 细分 | 无 | 中间精度 | Qdrant production |
| Qdrant Asymmetric | hybrid stored vs query | 无 | 磁盘 I/O bound | Qdrant production |
| **Weaviate RQ8 (NEW)** | **rotation + 8-bit scalar** | 无 (实证 good tradeoff) | **Weaviate default** | **Weaviate production** |
| **Weaviate RQ1 (NEW)** | **rotation + 1-bit binary** | 无 (cost-sensitive) | Weaviate optional | **Weaviate production** |
| **Weaviate PQ/BQ** (per llms.txt) | RQ/PQ/BQ all available | 无 | flexibility | Weaviate production |

### 三大 OSS vector DBMS quantization 对比（NEW）

| | Milvus | Qdrant | **Weaviate** |
|---|---|---|---|
| Default | 不强默认 | 显式可选 | **RQ8 default** |
| Available | PQ / SQ8 / ScaNN | Scalar / Binary / 1.5-2bit / Asymmetric / PQ | **RQ8 / RQ1 / PQ / BQ** |
| RaBitQ 集成 | ✗ | ✗ | ✗ (但 RQ 同源) |
| Rotation-based 思路 | n/a (PQ k-means) | n/a (Scalar 不 rotate) | **✓ RQ default** |
| Binary path 细分 | n/a | **1.5/2bit + Asymmetric** | RQ1 (1-bit) |

→ **Weaviate 是唯一 default rotation-quantization 的 OSS vector DBMS**——这与 RaBitQ 论文 2024 提出的 hypercube + random rotation 思路同源（但更早开始集成）。

### "rotation-quantization 范式" 在 production 的演化（NEW）

```
论文 (2024)         RaBitQ: 1-bit + theoretical O(1/√D) bound
                     ↓
Production OSS:
  Weaviate: RQ8 (4×, 8-bit) default + RQ1 (32×, 1-bit) optional
  Qdrant:   Asymmetric BQ (32× store + scalar query) — similar 思路
  Milvus:   未集成 rotation-based
```

→ Rotation-quantization 从论文到 production 已有 OSS 实现（Weaviate 是最显式的），但 **theoretical error bound (RaBitQ 主要贡献) 尚未进入任一 production DBMS**——是 wiki 内 frontier。

### 已知盲区

- **Weaviate RQ 是否与 RaBitQ 同算法**：docs 未提；可能 random rotation 部分同源但 quantization rule / estimator 设计不同
- **Weaviate RQ8 vs RQ1 实证 recall/speed**：docs 给"good tradeoff" 但未公开 benchmark
- **Weaviate vs Qdrant 同硬件 quantization head-to-head**：完全空白
- **production OSS vector DBMS 集成 RaBitQ 时间表**：未知

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
