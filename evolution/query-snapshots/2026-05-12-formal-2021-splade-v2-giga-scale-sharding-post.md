---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, systems/vespa.md, systems/weaviate.md, systems/turbopuffer.md]
cited-count: 5
---

# Post-snapshot (formal-2021-splade-v2): giga-scale-sharding

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 在 giga-scale workload 上提供 sparse path infrastructure 选择 — 与 dense path sharding 决策正交**——之前 wiki sharding 5 路径 (a-e) 全 dense-side. **关键 NEW**: production giga-scale retrieval 通常 dense (HNSW + ANN) + sparse (BM25 / SPLADE + inverted index) 两 path 并存, **sparse path 通常用 BM25-era 成熟的 inverted index 基础设施 (BlockMaxWAND, Anserini, Pyserini 等), 不需 vector DB native ANN-style sharding**. SPLADE FLOPS regularizer 直接优化 sparse path 查询成本 → giga-scale 下与 dense path 形成 cost balance: dense path expensive per-query, sparse path cheap per-query but lower recall — hybrid 最 cost-effective.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| 5 架构 (a-e) sharding | dense-side only | **不变, 仅 dense path 适用** |
| Sparse path sharding in giga-scale | 未涵盖 | **NEW: inverted index 成熟方案 (BlockMaxWAND, Pyserini), 不同 sharding axis 同 dense ANN** |
| Hybrid pipeline cost model | dense ANN only | **NEW: sparse cheap + dense expensive = production cost balance** |

### Sparse path 在 giga-scale 的不同 sharding 哲学

[per formal-2021-splade-v2 + IR production reality]

**Inverted index sharding (BM25/SPLADE 共通)**:
- Sharding by document ID range (Pyserini / Anserini 主流)
- Per-shard 独立 inverted index
- Query fanout to all shards → 每 shard 取 top-K → application 合并
- **Linear scale with corpus size**, no Asymptotic problem like dense ANN (P × log(|X|/P))

**与 dense ANN sharding 对比**:
- Dense ANN: complex routing (DistributedANN single graph / SPANN centroid / Pinecone slab / Milvus segment / Turbopuffer namespace)
- Sparse inverted: simple doc-ID hash sharding + parallel fanout
- **Sparse path 更适合大规模**—— inverted index 路由开销极低

### 16 节点 + 1TB RAM × 768-d hybrid workload 推算

[per formal-2021-splade-v2 production scale + Phase 1-2]

**Workload assumption (per query 原 query)**:
- 千亿规模 (100B docs)
- Dense path: 768-d MRL voyage-3 embedding + HNSW
- Sparse path: SPLADE encoded (~50 non-zero terms / doc)

**Storage**:
- Dense path: 100B × 768 × 1 byte (int8 post-MRL+QAT) = 76.8 TB / 16 nodes = 4.8 TB / node
- Sparse path: 100B × 50 terms × 8 bytes (term ID + impact) = 40 TB / 16 nodes = 2.5 TB / node
- **Total**: 7.3 TB / node SSD — feasible (5-10 TiB SSD/node typical)

**Latency**:
- Sparse path (inverted index lookup): ~ms (BlockMaxWAND-style fast)
- Dense path (HNSW + MRL prefix 256-d): ~数 ms
- Cross-encoder rerank top-100: ~10-30 ms (optional)
- Total: < 50 ms P99 ✓

**Throughput**:
- Sparse: ~10K QPS / node (CPU-bound on inverted index)
- Dense: ~5K QPS / node (HNSW + cosine)
- 16 nodes total ~ 80K-160K aggregate hybrid QPS

### 千亿/万亿决策表（updated 2026-05-12 post formal-2021-splade-v2）

| Workload | 推荐方案 |
|---|---|
| **千亿 hybrid retrieval (sparse + dense) + 复杂 ranking** | **Vespa**: BM25 + SPLADE + CLIP/MRL dense + rank-profile + global-phase cross-encoder rerank |
| 千亿 hybrid + AI-native primary DB | **Weaviate**: BlockMaxWAND BM25 + RQ8 dense + `hybrid()` first-class |
| 多租户 hybrid SaaS | **Turbopuffer**: namespace per tenant + multi_query (BM25 + dense) + application RRF |
| 千亿 single dense corpus + 6× throughput | DistributedANN-style (Microsoft Bing path, dense only) |
| 闭源 SaaS managed hybrid | Pinecone Sparse-Dense Hybrid Index |
| 千亿 + 多 index_type + OSS Go | Milvus sparse_inverted + 多 vector field |

### 已知盲区

- **Sparse path × dense path sharding cost balance**: production 实际 cost ratio (per-query CPU/memory/IO) 不公开
- **SPLADE inverted index at 100B+ scale**: 论文实测 MS MARCO 8.8M passages, giga-scale unknown
- **Vespa rank-profile 实际 giga-scale production case**: docs 提及但 case study zero
- **Cross-encoder rerank latency budget at 100B+ scale**: production case 不公开

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
