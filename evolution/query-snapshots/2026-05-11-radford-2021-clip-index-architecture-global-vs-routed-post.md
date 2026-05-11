---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/distributedann.md, systems/vespa.md, systems/turbopuffer.md]
cited-count: 5
---

# Post-snapshot (radford-2021-clip): index-architecture-global-vs-routed

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 不改变 5-架构 (a-e) 选择**——CLIP embedding 是 vector content 哲学层, 不影响 vector index 架构哲学 (single graph / partitioned / namespace-fanout / no-index-tenant / KV-store-shared-disk). **关键 NEW**: 但 CLIP-style multimodal workload 引入 **(e) namespace-as-architectural-primitive** 的新 motivation——**multi-CLIP-version coexistence** (consumer 用 ViT-B/32, B2B 用 ViT-L/14) **天然需要 namespace 级 isolation**, 强化 Turbopuffer 路径在 multimodal SaaS workload 的优势. **每 tenant 独立 CLIP variant** 是 Turbopuffer namespace-as-tenant 哲学的 multimodal-specific production motivation.

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| 5 架构 (a)-(e) | unchanged | **不变** |
| Multimodal-specific 架构动机 | n/a | **NEW: per-tenant 独立 CLIP variant 加强 (e) namespace-as-primitive 路径动机** |
| Vespa multi-tensor + 4-phase ranking | secondary feature | **NEW: 在 multimodal context 下是 first-order 决策驱动** |

### Multimodal 视角下 5 架构对照（NEW）

[per radford-2021-clip + wiki vendor multimodal support]

| 架构 | Multimodal-specific 适配 |
|---|---|
| (a) Global single index | **CLIP single-model single-corpus** 路径; DistributedANN single-graph (Bing-style) 在 multimodal large-scale 是 candidate (论文不明示 multimodal) |
| (c) 层次路由 | SPANN-style 适合 CLIP large-scale; Vespa SPANN OSS path multimodal-aware (tensor framework) |
| (d) 无索引 + tenant 分区 | Vespa Streaming + multi-tensor field; per-user multimodal RAG ideal |
| **(e) namespace-as-architectural-primitive** | **Turbopuffer namespace per CLIP variant: B2B SaaS 每客户独立 multimodal model**——multimodal-friendly architecture |

### Multimodal SaaS workload 的架构选择驱动（NEW insight）

[per radford-2021-clip variants + Turbopuffer namespace philosophy + Vespa application package]

E-commerce / B2B SaaS multimodal RAG 实际生产需求:
1. **多 CLIP variant 并存**: consumer 端 ViT-B/32 cost-friendly; B2B 客户 ViT-L/14 quality; specialized vertical 用 BiomedCLIP / RemoteCLIP / FashionCLIP
2. **多 model version**: CLIP base 与 fine-tune variant
3. **多语言 multimodal**: multilingual CLIP variant
4. **多 tenant 数据 isolation**: SaaS 安全合规

→ 这些需求**天然映射 namespace-as-primitive (e)**:
- 1 tenant = 1 namespace = 1 (CLIP variant, model version, language)
- Atomic deploy / migration per namespace
- Independent index parameters per namespace

→ Turbopuffer (100M+ namespace S3 prefix) + Vespa multi-tensor field 是 multimodal SaaS workload 最自然 architecture.

### 千亿规模 multimodal 主流选择

[per wiki architecture landscape]

之前 (text-only) 千亿规模主流: (a) DistributedANN 或 (c) Vespa SPANN. 加上 multimodal:
- (a) **DistributedANN multimodal**: Bing-style single-graph distributed; 论文不明示 multimodal, 但应可服务. 优势 6× throughput. 劣势: 不支持 multi-CLIP-version 自然 (single graph share embedding space)
- (c) **Vespa SPANN multimodal**: tensor framework + 4-phase ranking + SPANN; 同样 production-validated; multi-tensor field 支持 multi-CLIP-version. 缺点: latency penalty
- (e) **Turbopuffer multimodal**: namespace-as-tenant 完美支持 multi-CLIP-version; 但 single-corpus large-scale 不擅长 (per-namespace 500M doc 上限)

→ **千亿 multimodal corpus**: Vespa SPANN multi-tensor 或 DistributedANN-style (假设支持 multimodal). **千亿+ multi-tenant multimodal**: Turbopuffer namespace fanout.

### 架构选择决策表（updated 2026-05-11 post radford-2021-clip）

| 场景 | 推荐 |
|---|---|
| 千亿 single multimodal corpus + 复杂 ranking + ML rerank | **Vespa SPANN + multi-tensor + 4-phase ranking** |
| 千亿 single multimodal corpus + 单 CLIP variant + 6× throughput | DistributedANN-style (multimodal 不明示) |
| **多 CLIP variant SaaS B2B + per-tenant model** | **Turbopuffer namespace-as-tenant** |
| Multi-tenant multimodal personal AI (per-user ≤1M) | Vespa Streaming + multi-tensor field |
| Multimodal RAG 中等规模 (10B-100B) | SPANN/SPFresh path (Vespa OSS / Turbopuffer SaaS) |
| Multimodal HNSW + memory (≤1B) | 5 OSS vendor + Pinecone 任选 |

### 已知盲区

- **DistributedANN on multimodal**: paper 不明示 vector type
- **Per-tenant CLIP variant 实际 production routing**: 不公开
- **Multimodal SaaS large-scale production case (≥100B multimodal across multi-tenant)**: 不存在公开

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
