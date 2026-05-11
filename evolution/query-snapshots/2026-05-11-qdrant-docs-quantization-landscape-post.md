---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, concepts/product-quantization.md, concepts/rabitq.md]
cited-count: 3
---

# Post-snapshot (qdrant-docs): quantization-landscape

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 引入 wiki 内未见的 Binary path 细分**——除经典 PQ + Scalar 外，Qdrant 提供 **Binary Quantization (32× compression, 40× speedup)** + **1.5-bit / 2-bit BQ (中间精度)** + **Asymmetric Quantization (Binary store + Scalar query, 磁盘 I/O 友好)**——这些都是已有 quantization landscape 的工业实现细化。**关键观察**：Qdrant 没集成 RaBitQ（同年发表但论文层尚未触及生产实现），所以 Qdrant 的 quantization 仍属"无理论 error bound 的 heuristic family"——理论上 + RaBitQ 是 logical work。

## Answer

### Qdrant Quantization 多变体（NEW）

[per sources/docs/qdrant/manage-data/quantization.md]

| Method | Qdrant 版本 | Compression | wiki 已有 vs Qdrant |
|---|---|---|---|
| Scalar (SQ) | v1.1.0 | 4× | 同 Faiss SQ8 |
| Binary (BQ 1-bit) | v1.5.0 | **32× + 40× speedup** | wiki 仅 indirect 提及（Faiss survey）→ **Qdrant 是 production 实证 first-class** |
| **2-bit Quantization** | v1.15.0 | 16× | **wiki 未覆盖**——小维度更准 (-1/0/1 三 buckets) |
| **1.5-bit Quantization** | v1.15.0 | 24× | **wiki 未覆盖**——2-bit 折中 |
| **Asymmetric Quantization** | v1.15.0 | 32× store + scalar query | **wiki 未覆盖**——stored vs query 不同 encoding；磁盘 I/O bound 友好 |
| Product (PQ) | v1.2.0 | 16-32× | 同 Faiss IVFPQ |

### Quantization landscape（updated with Qdrant variants）

[per concepts/product-quantization.md "后续演化" + 新增 Qdrant variants]

| 方法 | 类型 | error bound | 主要 use case | wiki coverage |
|---|---|---|---|---|
| Binary / SQ8 | scalar | 无 | 内存极限 | indirect via Faiss |
| **Qdrant SQ** | scalar | 无 | Qdrant 默认压缩 | systems/qdrant.md |
| PQ / OPQ / LSQ | product | 无 | 减内存 | full |
| ScaNN anisotropic | score-aware | 无 | MIPS | full |
| VGPQ | PQ + Voronoi | 无 | OLAP (ADBV) | full |
| ACORN compression | graph edges | 无 | predicate-agnostic | full |
| **RaBitQ** | hypercube + random rotation | **O(1/√D) sharp w.h.p.** | 理论保证 quantization | full |
| **Qdrant Binary (1-bit)** | binary | 无 (rescore needed) | 32× + 40× speedup | systems/qdrant.md (NEW) |
| **Qdrant 2-bit (-1/0/1)** | binary 细分 | 无 | 小维度更准 | systems/qdrant.md (NEW) |
| **Qdrant 1.5-bit** | binary 细分 | 无 | 中间 trade-off | systems/qdrant.md (NEW) |
| **Qdrant Asymmetric (BQ store + SQ query)** | hybrid | 无 | 磁盘 I/O bound 优化 | systems/qdrant.md (NEW) |

→ Qdrant 在 **Binary path 三处细化**（2-bit / 1.5-bit / Asymmetric）是 wiki 已 ingest landscape 之外的工业实现深度。

### Qdrant 不集成 RaBitQ 的 gap（NEW）

[per concepts/rabitq.md + systems/qdrant.md "Open Questions"]

| Quantizer | Qdrant 实现 | RaBitQ 实现 | Status |
|---|---|---|---|
| Scalar (4×) | ✓ v1.1.0 | n/a | Qdrant only |
| Binary 1-bit (32×) | ✓ v1.5.0 (heuristic) | RaBitQ 1-bit = D bits, unbiased + sharp bound | **wiki 内不同范式** |
| Binary 1.5/2-bit | ✓ v1.15.0 (heuristic) | RaBitQ 未覆盖此粒度 | trade-off 不同方向 |
| Asymmetric | ✓ v1.15.0 (different stored vs query encoding) | RaBitQ B_q = 4-bit query (类似但 unbiased) | concepts 重叠但 Qdrant 是 production 实现 |

→ **Qdrant Binary + RaBitQ 理论 combine**：用 RaBitQ unbiased estimator + Qdrant Binary 1-bit compression——但两个工作不同论文 / 公司，未实证。Qdrant docs 不提 RaBitQ。

### Asymmetric Quantization 的特殊 insight（NEW）

[per sources/docs/qdrant/manage-data/quantization.md "Asymmetric Quantization"]

> **stored vectors 用 binary encoding** + **query vectors 用 scalar encoding** → **storage = binary, 但 distance compute 精度高于纯 binary**。Bottleneck disk I/O 时（多数 production）特别有用。

→ 这是**新的 quantization use case**：之前 wiki 隐含"store + query 同一 encoding"假设；Asymmetric 是"按 layer 配 quantization"——store 极致压缩 / query 极致精度。

### 已知盲区

- **Qdrant + RaBitQ 集成**：完全空白
- **Qdrant Binary 1.5-bit / 2-bit 在 production 实测**：docs 给"good accuracy with OpenAI / Cohere"但 wiki 内独立 verification 缺
- **Qdrant Asymmetric vs DiskANN-style PQ + SSD rerank**：两者都是"store low + query high"但实现 layer 不同（quantization vs storage tier）；对比未做
- **Qdrant + ScaNN anisotropic**：Qdrant 未集成 score-aware quantization
- **Qdrant + Filterable HNSW + Binary**：HNSW filter-aware build + 极致 quantization 联合实测

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
