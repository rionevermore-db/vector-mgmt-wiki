---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md, systems/distributedann.md, systems/turbopuffer.md, systems/milvus.md]
cited-count: 6
---

# Post-snapshot (radford-2021-clip): giga-scale-sharding

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 把 query 假设的 "向量维度 768" 落实为 production case**——ViT-L/14 输出 768-d 是 wiki 内**第一个明示的 768-d production multimodal embedding 案例**. **关键 NEW**: 768-d 不是任意工程数字, 而是 CLIP variant 的典型 production output. 此 query 假设的 16 节点 + 1TB RAM 拓扑下, 决策选择**不变**——giga-scale sharding 主要由 vector index algorithm (HNSW/SPANN/SPFresh/DistributedANN) 与 vendor 哲学决定; multimodal/CLIP 不引入新 sharding 路径. **但 CLIP-specific consideration**: production multimodal pipeline 同时需 (a) cosine ANN, (b) cross-modal hybrid query support, (c) multi-CLIP-version coexistence——这些需求**在 sharding 决策中作为 secondary requirement** 起 vendor differentiation 作用 (e.g., Vespa multi-tensor + tensor framework 比简单 cosine ANN vendor 更适合 multimodal giga-scale workload).

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| Sharding 5 路径 (a-e) | unchanged | **不变** |
| 768-d production 维度 case | 仅 query 假设 | **NEW: CLIP ViT-L/14 = 768-d 真实 production case** |
| Multimodal context 下 sharding 选择 | n/a | **NEW: tensor framework / multi-vector field / per-tenant model 成为 secondary criteria** |

### 768-d 维度 production 案例（NEW）

[per radford-2021-clip §2.4]

CLIP variant 维度落地:
- ViT-B/32: **512-d** (typical lower-cost production CLIP)
- ViT-B/16: 512-d (中等)
- ViT-L/14: **768-d** (paper "best" baseline)
- ViT-L/14@336px: 768-d (paper "best" full)
- RN50×16 / RN50×64: 768 / 1024-d

→ Query "向量维度 768" 直接对应 ViT-L/14 production case——multimodal retrieval workload, 不是任意工程数字.

### 16 节点 + 1TB RAM × 768-d CLIP-style multimodal workload

**Storage**:
- 1B 768-d float32 = 3 TB
- 1B 768-d int8 (4× compress) = 768 GB → fit 16 nodes × ~50 GB/node memory budget
- 100B 768-d int8 = 76.8 TB → 16 nodes × ~5 TB SSD/node (SPANN/DistributedANN posting on SSD path)
- 1T 768-d int8 = 768 TB → 16 nodes × ~50 TB SSD/node (超 SKU 上限) OR multi-slice approach

**Sharding 选择 (multimodal-aware)**:

| Workload tier | 推荐 |
|---|---|
| ≤10B 768-d CLIP @ 16 nodes | HNSW + cosine (Milvus/Qdrant/Weaviate/Vespa) |
| 10-100B 768-d CLIP | SPANN/SPFresh path (Vespa SPANN + Turbopuffer SPFresh) + int8 quantization |
| 100B-1T 768-d CLIP | DistributedANN single-graph distributed (Microsoft Bing-style) OR multi-slice partitioning |
| ≥1T 768-d CLIP | Bing-style multi-slice (50B per slice × N slices) OR Turbopuffer 100M+ namespace fanout |

**Multimodal-specific 加分项**:
- **Vespa**: tensor framework + ONNX inline CLIP encoder + multi-tensor field + 4-phase ranking → multimodal-friendly secondary criteria
- **Turbopuffer**: namespace-as-tenant → per-tenant CLIP variant 隔离 (e.g., consumer/B2B 用不同 CLIP variant)
- **Milvus**: 多 vector field + DiskANN/HNSW/CAGRA index 多选 → multimodal hybrid query 灵活
- **Pinecone**: Pinecone Inference 内置 CLIP-style embedding → 单 vendor 完整 multimodal pipeline (黑盒)

### 千亿/万亿 multimodal CLIP-style 决策表（updated 2026-05-11 post radford-2021-clip）

| Workload | 推荐方案 |
|---|---|
| **千亿 768-d CLIP + 复杂 ranking (LTR) + 4-phase rerank** | **Vespa SPANN + 4-phase ranking + tensor framework** |
| **千亿 768-d CLIP + 单一巨大 corpus + 6× throughput + 闭源接受** | **DistributedANN style (Microsoft Bing path)** |
| 多租户 multimodal (B2B SaaS, 每客户独立 CLIP variant) | **Turbopuffer namespace-as-tenant** |
| 千亿 768-d + 多 index_type + OSS Go | Milvus DISKANN/HNSW/CAGRA |
| 闭源 SaaS managed multimodal | Pinecone Inference + slab |
| Per-tenant 数据小, multimodal personal AI assistant | Vespa Streaming per-tenant + multi-tensor field |

### 已知盲区

- **768-d CLIP embedding 在 16-node 1TB RAM 实测**: 各 vendor 公开数据不存在
- **Multimodal hybrid query (CLIP + filter + BM25)**: 复杂 query 的 latency budget 在 giga-scale 下不公开
- **DistributedANN on multimodal**: Bing paper 不公开 vector type 是否包含 CLIP-style multimodal
- **CLIP fine-tune 频率 vs index rebuild cost**: production 案例不公开

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/milvus.md](../../systems/milvus.md)
