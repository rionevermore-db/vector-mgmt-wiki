---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, systems/vespa.md, systems/weaviate.md, systems/qdrant.md, systems/milvus.md, systems/pinecone.md]
cited-count: 7
---

# Post-snapshot (vespa-docs): quantization-landscape

## TL;DR (delta from weaviate-docs post)

**Vespa quantization 视角与 peer OSS DBMS 不同**——其他系统暴露 quantization 作为**索引参数**（HNSW + PQ / HNSW + RQ8）；Vespa 把它放到**tensor cell type 层**：`tensor<float>(x[768])` vs `tensor<int8>(x[768])` vs `tensor<bfloat16>(x[768])` vs **single-bit binary `tensor<int8>(x[96])` packed**——quantization 变成**schema-level 决策**而非索引参数。**关键 NEW**：Vespa 首次在 wiki 中明示**bfloat16 一等公民支持**（其他 OSS 仅 float32 + int8 + binary）+ **matryoshka-style nested embedding**（同一 doc 多分辨率向量共存）。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| Quantization 一等公民系统 | Qdrant (8-bit/binary/1.5-bit/2-bit/Asymmetric/PQ) + Weaviate (PQ/BQ/SQ/**RQ**) | **+ Vespa (cell type: float/double/bfloat16/int8/single-bit binary)** |
| Quantization 暴露层 | 索引参数 (Qdrant `quantization_config`) / vector index config (Weaviate) | **Vespa: schema-level tensor cell type** |
| bfloat16 一等公民 | ✗ | **✓ Vespa first** |
| Matryoshka-style nested embedding | ✗ | **✓ Vespa via `tensor<float>(i{},x[N])`** map of vectors |
| RaBitQ 工业集成 | ✗（仍仅 Milvus future） | **不变** |

### Vespa 的 quantization-as-schema 哲学（NEW）

[per sources/docs/vespa/llms-full.txt §Tensors §Vector Search Intro]

Vespa 不在索引参数层暴露 quantization，而在**tensor 类型层**：

```
field embedding type tensor<int8>(x[768])      # int8 量化，schema 固定
field embedding_bf16 type tensor<bfloat16>(x[768])  # bfloat16 量化
field embedding_bin type tensor<int8>(x[96])   # single-bit binary, 96 bytes pack 768 bit
```

→ **后果**：
1. **HNSW + quantization 不是"开启 flag"，而是 schema 选型**——切换 cell type 需要 application package 重新部署（atomic）
2. **混合 quantization 部署** trivial——一个 doc 多个 quantized representation 字段
3. **Matryoshka-style 多分辨率**：同 doc 可有 dense float32 + int8 + binary 三套，query-time YQL 选 tier

### Vespa 单 bit (binary) quantization（NEW）

[per sources/docs/vespa/llms-full.txt §Approximate Nn Hnsw]

Vespa `tensor<int8>(x[96])` packing 768-bit 进 96 bytes——native binary quantization。配合 **hamming distance** 算子：
- 内存：768 floats × 4 bytes = 3072 bytes → 96 bytes (**32× compress**)
- 距离：bitwise XOR + popcount，CPU SIMD 极快

→ Vespa 是 wiki 内**第二个 production 单 bit binary quantization 系统**（之前 Weaviate 也有 binary 但通过 RQ 8-bit + BQ 1-bit 系列），Vespa 直接 cell type 层 promote。

### Quantization landscape 全景表（updated 2026-05-11 post vespa-docs）

| 方法 | 压缩率 | 精度损失 | 速度 | 工业 production 出现 |
|---|---|---|---|---|
| Float32 (baseline) | 1× | 0 | baseline | 全部 |
| **bfloat16** | 2× | 极小 | ~1.2× | **Vespa first-class** |
| Float16 (FP16) | 2× | 极小 | ~1.2× | Weaviate (option) |
| Scalar Quantization (int8) | 4× | 小 | 1.5-2× | Qdrant default + Weaviate SQ + **Vespa cell type** |
| **RQ8** (random rotation + scalar) | 4× | 比 SQ8 小 | 1.5-2× | Weaviate default |
| RQ1 / Binary (1-bit) | 32× | 中等 | 5-10× | Qdrant BQ + Weaviate BQ + **Vespa single-bit binary** |
| Qdrant 1.5-bit / 2-bit | 16-21× | 介于 SQ 和 BQ | 3-6× | Qdrant only |
| PQ (8x8 = 64-bit) | 24-48× | 中 | 2-3× | Faiss + Milvus + Pinecone + Weaviate option |
| OPQ | 同 PQ | 比 PQ 小 | 2-3× | Faiss + Milvus IVF_PQ (option) |
| **RaBitQ** | 32× (1-bit) | **unbiased + sharp bound** | 3× (per Faiss) | 仍仅 Milvus future + Faiss research |

### 工业组合策略（updated）

- **Vespa**: HNSW + cell type quantization (schema-level)，**唯一支持 matryoshka 多分辨率** in schema
- **Weaviate**: HNSW + RQ8 default / BQ / PQ option
- **Qdrant**: HNSW + 8-bit (default) / BQ / 1.5-bit / 2-bit / Asymmetric / PQ
- **Milvus**: HNSW/IVF/DISKANN/CAGRA + IVF_PQ / IVF_SQ8 / SCANN（PQ 家族 dominant）
- **Faiss**: 全栈（PQ / OPQ / SQ / RaBitQ research）
- **Pinecone**: 黑盒 slab adaptive，外部无法验证

### 已知盲区

- **bfloat16 实测 recall/latency**：Vespa 一等公民但 docs 未量化与 float32 / int8 的具体差异
- **Single-bit binary head-to-head**：Vespa vs Qdrant BQ vs Weaviate BQ 实测
- **RaBitQ production**: 仍无 OSS DBMS 集成
- **Matryoshka 多分辨率 ranking**: Vespa 提供基础设施但 query 阶段 tier-pick 策略论文/docs 未深入

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
