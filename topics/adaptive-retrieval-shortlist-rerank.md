---
title: Adaptive Retrieval（多粒度 shortlist + rerank）
type: topic
sources: [kusupati-2022-matryoshka, vespa-docs, turbopuffer-docs]
related: [../concepts/matryoshka-embedding.md, ../concepts/clip.md, ../concepts/product-quantization.md, ../concepts/rabitq.md, ../concepts/hnsw.md, ../systems/vespa.md, ../systems/turbopuffer.md, ../systems/milvus.md, ../systems/qdrant.md, ../systems/weaviate.md, ../systems/pinecone.md, ../systems/distributedann.md, multimodal-embedding-retrieval.md, index-selection.md, topk-vs-iterator-model.md, multi-vector-queries.md]
created: 2026-05-11
updated: 2026-05-11
---

# Adaptive Retrieval（多粒度 shortlist + rerank）

**TL;DR**: Adaptive Retrieval (AR) 是把 ANN query 拆成 **multi-stage 多粒度 pipeline** 的 retrieval pattern: 用**低维 / 量化 representation 做 shortlist** (cheap, large recall), 用**高维 / 全精度 representation 做 rerank** (expensive, precise top-K). **wiki 内已被多个 vendor production 部署但未系统化**——CLIP 的 prompt ensembling + Cohere/Voyage cross-encoder rerank / Vespa 4-phase ranking / DistributedANN head-index 起点 + global-graph rerank / Faiss IVF + flat rerank——**所有都是 AR 的不同实例化**. **Matryoshka Representation Learning** [kusupati-2022-matryoshka] 把 AR 从工程 trick 升级为**有数学保证的 retrieval primitive**: 单一 MRL-trained embedding 内**128× FLOP / 14× wall-clock 加速 with comparable mAP@10**——shortlist 用 16-d prefix, rerank 用 2048-d 全维. **关键 wiki 含义**: 之前 AR 的多种实例化 (cascade quantization / coarse-fine routing) 都需要 **trained-time 介入** (PQ retrain / 多 model train); MRL 让 **single-model + dim truncation** 就能实现 AR. **下一代 production retrieval pipeline** = MRL backbone + 1 个 vector DB schema + AR pipeline.

## 问题陈述

[per kusupati-2022-matryoshka §4.3 + production retrieval reality]

ANN retrieval 单 query 的 cost model:
- **Search cost**: O(d × N) for exact, O(d × log(N)) for HNSW
- **d** 是 embedding 维度 (典型 768-2048)
- **N** 是 corpus size (典型 10M-1B)

减 d (vector compression) 和减 N (partition + pruning) 是 **正交两个 axis**——AR 同时利用:

1. **减 d in stage 1** (shortlist): 16-d ANN over N candidates → 取 top-K (K=200) candidates with low cost
2. **不减 N in stage 2** (rerank): 2048-d exact over K candidates → top-K' (K'=10) final

stage 1 节省 (d_full - d_short) / d_full × search_cost = (2048 - 16) / 2048 ≈ 99.2% search FLOPs.
stage 2 重新 score K=200 candidates 是 K × d_full = 200 × 2048 = 400K FLOPs (trivially cheap).

**总 speedup**: (d_full / d_short) × ANN_speedup ≈ 128× theoretical, **14× wall-clock** real-world (per [kusupati-2022-matryoshka §4.3.1 Figure 8]).

## 相关概念

- **[Matryoshka Embedding](../concepts/matryoshka-embedding.md)**: AR 的算法基础——一次训练 log(d) 个 nested prefix, vector DB 端 0-cost truncate
- **[Product Quantization](../concepts/product-quantization.md)**: AR 早期 cascade form (IVF + flat rerank)——PQ 做 shortlist, exact rerank
- **[RaBitQ](../concepts/rabitq.md)**: AR 的现代实例——binary rerank + error bound 判断 trigger condition
- **[HNSW](../concepts/hnsw.md)**: AR shortlist stage 的 default graph index
- **[CLIP](../concepts/clip.md)**: cross-modal AR 输入 (text → image AR with prompt ensembling)
- **[topics/multi-vector-queries.md](./multi-vector-queries.md)**: AR 一种特例 (one vector type for shortlist, another for rerank)
- **[topics/topk-vs-iterator-model.md](./topk-vs-iterator-model.md)**: AR 是 iterator pattern 的多 stage form

## 工业方案对比

### AR 的 5 种 production 实例化

[per wiki 内 vendor + paper + production retrieval reality]

| Variant | Shortlist | Rerank | Trigger | Production deployment |
|---|---|---|---|---|
| **MRL prefix truncation** | low-d prefix (e.g., 16-d) ANN | full-d (2048) exact | always | OpenAI emb-3 / Cohere v3-v4 / Voyage / Qwen3-VL / **Vespa matryoshka cell** |
| **IVF + flat rerank** ([PQ](../concepts/product-quantization.md)) | PQ code + IVF inverted list | exact full-precision | configurable nprobe | Faiss / Milvus IVF_PQ |
| **DiskANN-style PQ DRAM + SSD full** | PQ navigation in RAM | exact full vector on SSD | beam search | DiskANN / Microsoft / Milvus DISKANN |
| **SPANN-style centroid + posting** | in-mem centroid index | on-SSD posting list exact | top-N centroids | SPANN @ Vespa / SPFresh @ Turbopuffer |
| **Vespa 4-phase ranking** | retrieval (HNSW/BM25/weakAnd) | first-phase → second-phase → global-phase ONNX | rank-profile triggered | Vespa |
| **DistributedANN head + global** | in-mem head index 2.5B vec | full DiskANN graph traverse | beam search | DistributedANN @ Bing 2025 |
| **Cross-encoder cascade rerank** | bi-encoder ANN | cross-encoder LLM rerank | configurable top-K | Cohere Rerank / Voyage / mxbai / BGE-Reranker (application 层) |
| **Funnel retrieval (MRL)** | cascade dims 16→32→64→128→256→2048 | each stage halves shortlist | progressive | paper 提, 工业 case TBD |

### 5 个 AR 实例化的本质相同

[per kusupati-2022-matryoshka §4.3 + wiki retrieval pattern]

所有 AR 都满足 4 个性质:
1. **Stage 1 cheap**: low-d / quantized / inverted / clustered representation, ANN cost 远低 stage 2
2. **Stage 2 precise**: high-d / full-precision representation, exact distance compute
3. **Stage 1 → Stage 2 candidate set 缩小**: top-K' << corpus, rerank cost 仅 K' × d_full
4. **End-to-end recall ≈ single-stage full**: shortlist 不损 final top-K recall

差异在于 **stage 1 representation 选择**:
- MRL prefix: 训练时 explicit nested
- PQ code: post-hoc subspace quantization
- IVF centroid: clustering + posting list
- Cross-encoder: fundamentally different model

→ **MRL 是 AR 的"training-time first principle"; 其他都是 post-hoc 实例化**.

## 各 wiki vendor 的 AR 支持

[per wiki 内 6 vendor + DistributedANN]

| Vendor | Native AR pipeline | MRL-aware? | Cross-encoder rerank? |
|---|---|---|---|
| **Vespa** | **✓ 4-phase ranking (retrieval → first-phase → second-phase → global-phase ONNX)** | **✓ matryoshka cell type** (单 field 多 prefix 共存) | ✓ via global-phase ONNX/cross-encoder |
| Milvus | ✗ (HNSW + IVF + DISKANN 路径但应用层 rerank) | application 层 truncate prefix | application 层 |
| Qdrant | ✗ (HNSW + quantizer) | application 层 | application 层 |
| Weaviate | ✗ (HNSW + reranker via Inference module) | application 层 | ✓ via Reranker module |
| Pinecone | 黑盒 (slab adaptive, 不公开) | 不公开 | ✓ via Rerank API |
| Turbopuffer | ✗ ("focused on first-stage retrieval") | namespace per MRL variant | **明示推到 application** (search.py / search.ts) |
| DistributedANN | **✓ head-index + global-graph 2-stage** | paper 不明示 MRL | ✗ (paper 不涵盖 rerank) |

→ Vespa native 多 stage AR 最深; 其他 vendor 把 second stage 推到 application. **MRL 让所有 vendor 都能轻量级支持 AR (单 schema + 应用层 prefix query)**——不需要 vendor native MRL-aware.

## 历史 AR 路径与未来

[per kusupati-2022-matryoshka §1-2 + wiki history]

**Pre-2022 AR 实例化**:
- 1980s-1990s: IR cascade re-ranking (BM25 candidate → 复杂 ranker)
- 2010s: Faiss IVF + flat rerank
- 2010s: BM25 + dense vector hybrid (BM25 shortlist + dense rerank)
- 2019-2020: DiskANN PQ + SSD rerank, SPANN centroid + posting

**MRL 之后 (2022+)**:
- **production embedding models 默认 MRL training**: OpenAI / Cohere / Voyage / Snowflake / Mixedbread / Qwen 等
- **vector DB schema 简化**: 单一 full-d embedding store, AR 由 query path 自动决定
- **Vespa "matryoshka tensor" cell type 命名**: 直接来自此 paper
- **Turbopuffer QAT philosophy**: MRL-trained model 的 int8 输出与 f16 namespace 完美 fit (model side quantization + DBMS-side truncation 双 train-time technique)

**未来 frontier (per kusupati-2022-matryoshka §6 future work)**:
1. MRL-aware ANN index (learnable k-d tree)
2. Dim-specific loss (high-recall for 8-d, robustness for 2048-d)
3. Pareto-optimal `c_m` weighting
4. Joint MRL + searchable data structure

## Open Questions

- **AR 多 stage trigger condition optimization**: shortlist K, rerank top-K' 选择当前 ad-hoc; production case 是否需 workload-aware tuner? 不公开
- **MRL + IVF + flat rerank 三层 cascade**: full Funnel 实测在 ImageNet-1K 之外 production case zero coverage
- **Cross-encoder rerank cost vs MRL stage 2 cost trade-off**: cross-encoder LLM (Cohere Rerank-v3 等) 比 MRL full-d cosine rerank 慢 100×+ 但 quality 更高——production 何时切换? 不公开
- **AR 在 spatial / multimodal 三模 query 的 generalization**: shortlist 是否能用 spatial filter + MRL prefix combined? 不存在公开 case
- **AR 在 long-context retrieval**: 长文档 chunking + MRL prefix shortlist + 全 chunk rerank, 但 long-context production case (e.g., 100K token doc) AR pattern 不公开
- **AR vs single-stage HNSW 的 recall guarantee**: paper §4.3 给 mAP@10 comparable, 但 recall@1 / recall@5 (更严 metric) 在 AR 下退化曲线? 不公开
- **Vendor native AR support gap**: Milvus / Qdrant / Weaviate / Turbopuffer 都把 second stage 推到 application——native pipeline 何时成为 vendor 一等公民? Vespa 已 native 但其他 vendor 落后. industry trend 推测向 Vespa 哲学收敛
- **MRL + DistributedANN single graph integration**: paper 不明示 single-graph distributed 是否支持 MRL prefix; Bing 50B 单图 query 是否能用 MRL shortlist 加速? Open
- **AR + LLM-augmented retrieval**: agentic retrieval (Weaviate Query Agent, Pinecone Assistant) 的 multi-stage AR 协调? wiki 内 zero coverage
