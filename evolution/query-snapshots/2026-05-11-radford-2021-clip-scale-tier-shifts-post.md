---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md, systems/distributedann.md, systems/turbopuffer.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (radford-2021-clip): scale-tier-shifts

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 不引入新 tier**——但 wiki 内**embedding dimension axis** 从一般描述变为 production-specific 维度: ViT-B/32 = 512-d, ViT-L/14 = 768-d, ImageBind = 1024-d. **关键 NEW**: tier-shift 视角下, **embedding dim 是 vendor-orthogonal storage cost 维度**——dim 翻倍直接 storage 翻倍 (10B × 768 vs 10B × 1024 差 33%). **Multimodal embedding 的 production scale 上限 case**: CLIP-style multimodal retrieval 在 wiki 内**没有任何 vendor 公开 ≥10B production multimodal case**——multimodal frontier 在 large-scale tier (≥100B) 仍是 wiki 重要 coverage gap.

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| Tier 5 (≥1T) production case | Bing DistributedANN + Turbopuffer 3.5T+ | **不变** |
| Multimodal tier ceiling | 未涵盖 | **NEW: 无 vendor 公开 ≥10B multimodal production case** |
| Embedding dim → cost axis | 隐含 | **NEW: CLIP 512/768/1024-d 落地真实 dim** |
| Sharding routing 路径 (a-e) | 5 路径 | **不变** |

### Tier-shift 表与 embedding dim 的关系（NEW axis）

[per radford-2021-clip CLIP variant dim + wiki tier-shift]

| Tier (vector count) | Embedding dim 影响 | Multimodal-specific |
|---|---|---|
| Tier 1 (≤1B) | 512 vs 768 vs 1024 storage 差 50%-100%; HNSW + memory all-in | CLIP multimodal default path |
| Tier 2 (1-10B) | 768-d 全内存边界 ~22 TiB float32; 切 quantization | Multimodal RAG / 商品检索主流 |
| Tier 3 (10-100B) | 768-d × 100B × int8 = 76 TB SSD 路径强制 | **No vendor 公开此 tier multimodal production case** |
| Tier 4 (100B-1T) | DistributedANN (Bing 50B per slice) OR multi-slice | Bing DistributedANN 实际 vector type 不明示 multimodal |
| Tier 5 (≥1T) | Bing + Turbopuffer 2 数据点 | 无公开 multimodal case (Bing 50B 是 web doc + image-vector? 论文不细谈) |

### Multimodal-specific tier-shift 维度（NEW）

[per radford-2021-clip § + wiki industry coverage gap]

之前 tier-shift 仅看 **scale × storage × parallel**. CLIP ingest 引入第 4 维度: **modality complexity**:

1. **Single modality (text-only e.g., BGE / Voyage)**: 传统 wiki tier-shift 完全覆盖
2. **Dual modality (CLIP text+image)**: wiki 内 vendor 全 support 但 scale 上限不明示
3. **Multi-modality (ImageBind 6 modality, 1024-d)**: 是否能 fit 同 cosine ANN? **wiki zero coverage**
4. **Domain-specific multimodal (BiomedCLIP, RemoteCLIP)**: production routing 需求 vs general CLIP——架构维度新需求

→ **Tier-shift × modality complexity** 是 wiki 内 **embedding-side 新维度**——之前 tier-shift 假设 single modality (text)。CLIP 揭示 production 实际多 modality 但 wiki tier-shift 表未扩展.

### Tier-shift 决策驱动（updated 2026-05-11 post radford-2021-clip）

1. **Tier 1 (≤1B) text + multimodal both**: HNSW + memory 默认
2. **Tier 2 (1-10B) text-only**: 加 quantization; multimodal CLIP 也同
3. **Tier 3 (10-100B) text-only**: SPANN / DiskANN / SPFresh / Starling 路径; **multimodal CLIP 同 path 但实测公开 zero**
4. **Tier 4 (100B-1T) text-only**: DistributedANN / Bing-style multi-slice; **multimodal scale 不公开**
5. **Tier 5 (≥1T) text-only**: Bing + Turbopuffer 2 数据点; **multimodal ≥1T 完全空白**

### 关键 frontier: multimodal large-scale production 实证缺失

[per wiki industry coverage]

- Vespa multimodal 强能力但具体 production scale 不公开 (Yahoo 内部应用未明示 scale)
- Milvus multimodal via 多 vector field 但 100M+ multimodal benchmark 不存在
- Turbopuffer 提到"Cohere image model, Voyage image model 多模态扩展"但 scale 不明
- Pinecone Inference CLIP-style 但 closed-source
- **Bing DistributedANN 50B vector 是否含 multimodal? paper 不公开 vector type**

→ **Multimodal large-scale production frontier 是 wiki 内 most important next data point**——某 vendor 公开 ≥10B CLIP-style multimodal production case 将填补这个 critical gap.

### 已知盲区（updated 2026-05-11 post radford-2021-clip）

- **Multimodal large-scale production case (≥10B)**: 不存在 wiki 内任何 vendor 公开
- **Embedding dim 升级 (512→768→1024) tier-shift 成本**: 不公开
- **6 modality (ImageBind) 在 vendor 端实测**: 完全空白
- **Bing DistributedANN 50B 是否 multimodal**: 论文 §1 不明示 vector type, 推测主要 text doc

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
