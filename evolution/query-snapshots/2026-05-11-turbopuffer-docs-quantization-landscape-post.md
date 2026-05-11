---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, systems/turbopuffer.md, systems/vespa.md, systems/qdrant.md, systems/weaviate.md]
cited-count: 6
---

# Post-snapshot (turbopuffer-docs): quantization-landscape

## TL;DR (delta from vespa-docs post)

**Turbopuffer 是 wiki 内首个明示"quantization 弱化优先级"的 production system**——commercial SaaS 不暴露 quantization 选项，仅 `f16` vs `f32` vector dtype（performance 建议提到），quantization burden 留给"quantization-aware embedding model"（voyage-4/embed-v4/Qwen3-VL-Embedding-8B），由 application 层选 int8/f16 输出，传给 Turbopuffer namespace。**关键 NEW**：Turbopuffer 推一种**新 quantization 模式**——`int8 values directly as JSON integers to an f16 namespace`，即 quantization 是 **embedding model side concern**，DBMS 只暴露 cell type 接口；这是与 Qdrant / Weaviate / Milvus 把 quantization 作 index 参数完全相反的哲学。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| Quantization 一等公民系统 | Qdrant / Weaviate / Vespa | **+ Turbopuffer (但弱化)——f32/f16 cell type, 不暴露 PQ/SQ/BQ/RQ** |
| Quantization 暴露层 | 索引参数 (Qdrant/Weaviate) / schema cell type (Vespa) | **+ Turbopuffer: embedding model layer concern, DBMS 仅 cell type** |
| QAT (quantization-aware training) production 推荐 | wiki 未明示 | **Turbopuffer 文档明示 voyage-4 / embed-v4 / Qwen3-VL 等 QAT model** |
| RaBitQ production | 仍仅 future | **不变** |

### Turbopuffer "quantization is model-layer" 哲学（NEW）

[per sources/docs/turbopuffer/llms-full.txt §performance §write]

> For models with quantization-aware training (voyage-4 series, voyage-context-3, embed-v4, Qwen3-VL-Embedding-8B), `int8` output matches `f32` precision (benchmarks), **so you can pass `int8` values directly as JSON integers to an `f16` namespace for `f16` speed with no precision loss.**

→ 这是 wiki 内**首个把 quantization burden 完全推给 embedding model side** 的 production system。Turbopuffer 不提供：
- ✗ Product Quantization (PQ) 选项
- ✗ Scalar Quantization (SQ) 选项
- ✗ Binary Quantization (BQ) 选项
- ✗ RaBitQ option

仅提供：
- ✓ `f32` / `f16` namespace cell type
- ✓ "Pass int8 from QAT model → f16 namespace" 工程组合

**Trade-off**：
- 优势：DBMS 简洁，client 灵活选 best-of-breed embedding model + quantization
- 劣势：DBMS 内部不能再做 quantization → 比 Qdrant 1-bit BQ (32× compress) 内存优势小

### 工业 quantization landscape 全景表（updated 2026-05-11 post turbopuffer-docs）

| 方法 | 压缩率 | 精度损失 | DBMS-side vs Model-side | 工业 production 出现 |
|---|---|---|---|---|
| Float32 (baseline) | 1× | 0 | n/a | 全部 |
| **bfloat16** | 2× | 极小 | DBMS-side | Vespa first-class cell type |
| Float16 (FP16) | 2× | 极小 | DBMS-side | Weaviate option + **Turbopuffer namespace cell type** |
| Scalar Quantization (int8) | 4× | 小 | DBMS-side OR **Model-side via QAT** (Turbopuffer 哲学) | Qdrant default + Weaviate SQ + Vespa cell type + **Turbopuffer 哲学** |
| **RQ8** (rotation + scalar) | 4× | 较 SQ8 小 | DBMS-side | Weaviate default |
| RQ1 / Binary (1-bit) | 32× | 中等 | DBMS-side | Qdrant BQ + Weaviate BQ + Vespa single-bit |
| Qdrant 1.5/2-bit | 16-21× | 介于 SQ/BQ | DBMS-side | Qdrant only |
| PQ (8x8 = 64-bit) | 24-48× | 中 | DBMS-side | Faiss + Milvus + Pinecone + Weaviate option |
| OPQ | 同 PQ | 较 PQ 小 | DBMS-side | Faiss + Milvus IVF_PQ |
| **RaBitQ** | 32× (1-bit) | unbiased + bound | DBMS-side (research) | 仅 Faiss + Milvus future |
| **QAT model output (int8 → f16)** | 8× (f32 → int8 + f16 namespace) | 0 (per Voyage benchmark) | **Model-side** | **Turbopuffer 推荐 voyage-4/embed-v4/Qwen3-VL** |

### Turbopuffer 路径的工业组合

- Application: 调 voyage-4 / embed-v4 / Qwen3-VL 等 QAT model 获 int8 output
- Turbopuffer: namespace `f16` cell type, vectors 以 int8 JSON 写入（auto cast to f16）
- 净效果: **8× compress + 0 precision loss + Turbopuffer 端 0 quantization 配置**

vs peer DBMS 路径：
- Qdrant: HNSW + `quantization_config: scalar / binary / 1.5-bit / 2-bit / asymmetric`
- Weaviate: HNSW + `vectorIndexConfig: vectorIndexType + pq/bq/sq/rq`
- Milvus: index_type IVF_PQ / IVF_SQ8 / SCANN / HNSW_PQ
- Vespa: `tensor<int8>(x[768])` / `tensor<bfloat16>` / single-bit pack
- **Turbopuffer**: 只有 `f32` / `f16`——其他全推给 embedding model

### 工业组合策略（updated 2026-05-11 post turbopuffer-docs）

- **Turbopuffer + QAT model** (voyage-4 / embed-v4): int8 from model → f16 namespace, 0 quant config
- **Vespa + matryoshka**: 多 cell type 共存 + schema-level switch
- **Weaviate**: HNSW + RQ8 default / BQ / PQ option
- **Qdrant**: HNSW + 8-bit / BQ / 1.5-bit / 2-bit / Asymmetric / PQ
- **Milvus**: HNSW/IVF/DISKANN/CAGRA + IVF_PQ / IVF_SQ8 / SCANN
- **Faiss**: 全栈
- **Pinecone**: 黑盒 slab adaptive

### 已知盲区

- **QAT model int8 vs DBMS-side SQ8 在 production 实测对比**：Voyage 提供 benchmark，第三方 head-to-head wiki zero
- **Turbopuffer f16 namespace + int8 写入的内部转换 overhead**：docs 不细谈
- **Turbopuffer 是否未来加 DBMS-side quantization**：闭源 roadmap 不公开
- **QAT 之外 embedding model + DBMS quantization 重叠**：non-QAT model + DBMS BQ 是否有 stack？wiki 未深入

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
