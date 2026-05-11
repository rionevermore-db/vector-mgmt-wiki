---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, concepts/clip.md, topics/multimodal-embedding-retrieval.md]
cited-count: 4
---

# Post-snapshot (radford-2021-clip): quantization-landscape

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 显示 quantization landscape 的一个重要 missing axis**: **CLIP embedding 的 quantization tolerance 实测 wiki + 论文都 zero coverage**——CLIP 训练后 embedding 几何 (L2-normalized + cosine space) 在 OPQ/RaBitQ/Binary/PQ 各 quantizer 下 recall 退化曲线**不存在公开数据**. **关键 NEW**: production multimodal retrieval workload (Vespa / Milvus / Qdrant / Weaviate 等) 普遍 quantize CLIP embedding 来降本——但 quantization choice 对 CLIP cross-modal recall 的影响 (text→image, image→text) **可能与 text-only embedding 不同**——CLIP joint embedding space 同时 anchor 两种 distribution, quantization 损失可能不对称影响两种 retrieval direction.

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| OPQ production case | Faiss + Milvus + DistributedANN @ Bing 50B | **不变** |
| CLIP embedding quantization tolerance | n/a | **NEW: 实测 zero coverage despite production usage** |
| Multimodal embedding quantization asymmetry | n/a | **NEW: text→image vs image→text recall 不一定对称受 quantization 影响** |
| QAT multimodal model | Turbopuffer 提 voyage-multimodal-3 | 不变 |

### CLIP embedding 在 quantization 视角下的 production reality

[per radford-2021-clip embedding + wiki vendor quantization]

CLIP embedding 默认是 **float32 768-d** (ViT-L/14). Production storage cost:
- ViT-L/14 raw: 3072 bytes/vec → 1B vector = 3 TB
- 任何 quantization 都直接 4×-32× compress

各 vendor 服务 CLIP 的 quantization 路径:
- **Vespa**: cell type `tensor<int8>(x[768])` (4× compress) / `tensor<bfloat16>(x[768])` (2× compress) / single-bit binary `tensor<int8>(x[96])` packed (32× compress)
- **Qdrant**: HNSW + scalar/binary/1.5-bit/2-bit/asymmetric/PQ quantizer
- **Weaviate**: HNSW + RQ8 default / BQ / PQ / SQ options
- **Milvus**: HNSW/IVF_PQ/IVF_SQ8/SCANN
- **Pinecone**: 黑盒 slab adaptive (不公开是否 quantize CLIP)
- **Turbopuffer**: f32 / f16 cell type, quantization 推到 embedding model side (QAT multimodal model)

### CLIP-specific quantization 考量（NEW insights）

[per radford-2021-clip §2.4 + cross-modal retrieval reality]

1. **Cross-modal asymmetry hypothesis**: CLIP joint space 由 image + text 两 distribution 共同 train. PQ subspace 假设 features 独立——可能对 image features 更有效, text 一面 features distribution 不同导致 PQ subspace 在 text 一面 less optimal? **论文不验证**.

2. **L2-normalized + cosine space 上的 binary quantization**: CLIP embedding 在 unit sphere 上. Binary quantization (1-bit per dim) 大致保留 angle (cosine) 但 magnitude 信息丢失——但 magnitude 在 L2-normalized embedding 上本就是 1, **binary 实际不损 cosine 信息**——理论上 CLIP embedding 对 binary quantization 比一般 dense embedding 更 robust. **wiki / 论文都不实测**.

3. **OPQ 在 CLIP embedding 上 known 工作**: DistributedANN paper d_OPQ=64 over d=384 int8 (6× compress)——但 DistributedANN paper 没明示是否 multimodal. CLIP-specific OPQ ablation 不存在公开.

4. **RaBitQ 在 multimodal 上未实测**: RaBitQ unbiased + sharp bound 性质应该平移到 CLIP, 但 wiki 内 RaBitQ 仅 Faiss + Milvus future, **multimodal production 案例零**.

### Quantization landscape 全景表（updated 2026-05-11 post radford-2021-clip）

| 方法 | 压缩率 | CLIP-specific 适配性 | Multimodal production 出现 |
|---|---|---|---|
| Float32 (baseline) | 1× | trivial | 全部 |
| bfloat16 | 2× | 极小 loss (CLIP 训练用 mixed-precision) | Vespa cell type, Turbopuffer namespace |
| Float16 (FP16) | 2× | 极小 loss | Vespa / Weaviate / Turbopuffer |
| Scalar Quantization (int8) | 4× | **强 candidate** (CLIP normalize 后 distribution 较稳) | Qdrant default + Weaviate SQ + Vespa cell + Turbopuffer 哲学 |
| RQ8 (rotation + scalar) | 4× | 等效 SQ | Weaviate default |
| Binary (1-bit) | 32× | **理论上 CLIP cosine 距离保留较好** (unit sphere) | Qdrant BQ + Weaviate BQ + Vespa single-bit |
| PQ (8x8 = 64-bit) | 24-48× | **subspace 独立假设可能在 multimodal joint space 上 less optimal** | Faiss + Milvus + Pinecone + Weaviate option |
| OPQ | 同 PQ | **学习 rotation 部分修复 subspace mismatch**——理论上比 PQ 更适合 CLIP | DistributedANN @ Bing (uncertain multimodal) |
| RaBitQ | 32× (1-bit) | **unbiased + bound 适合 CLIP**——但实测案例零 | 仅 Faiss + Milvus future |

### 已知盲区（updated 2026-05-11 post radford-2021-clip）

- **CLIP embedding × 6 quantizer × 6 vendor × 3 retrieval direction (text→image, image→text, image→image)** 全 24+ combination 实测对比**完全不存在**——**wiki 内最重要 multimodal quantization frontier**
- **CLIP multimodal joint space PQ subspace independence violation**: 是否 image-domain subspace 与 text-domain subspace 应分别 quantize? 论文 / wiki 都不涵盖
- **CLIP variant (ViT-L/14 768-d vs ViT-B/32 512-d) quantization tolerance 差异**: 不同模型 size 的 quantization sensitivity 是否同?
- **Multimodal QAT model (voyage-multimodal-3 / Cohere multimodal) int8 输出在 vector DB 端的 production 实证**: Turbopuffer 推荐但具体 multimodal QAT 实测数据不公开

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
