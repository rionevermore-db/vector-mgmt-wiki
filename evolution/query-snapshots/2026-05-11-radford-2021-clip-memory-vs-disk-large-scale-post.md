---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/distributedann.md, systems/turbopuffer.md, topics/disk-vs-memory-ann.md]
cited-count: 5
---

# Post-snapshot (radford-2021-clip): memory-vs-disk-large-scale

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 不改变 memory vs disk 路径选择**——CLIP embedding 在 5 disk tier (HNSW memory / SPANN-DiskANN local NVMe / Vespa Streaming disk-only / Turbopuffer object storage / DistributedANN distributed KV) 中**transparent service**: 任何 tier 都 cosine ANN over normalized embedding. **关键 NEW**: 但 CLIP embedding 维度 (768-d ViT-L/14) 比传统 wiki benchmark (SIFT 128-d) 大 6×, **direct storage cost 6×**——这使得 large-scale tier 切换提前: 1B SIFT 在 memory 仍合理, 1B CLIP 在 memory 已撞墙 (3 TB). **Multimodal large-scale 实际 production path** 至少需 quantization (CLIP int8 → 768 GB 1B) OR disk path (SPANN/DiskANN/DistributedANN).

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| 5 disk tier (memory / NVMe / object storage / brute-force disk / distributed KV) | unchanged | **不变** |
| CLIP embedding 在各 disk tier 上的 cost | n/a | **NEW: 768-d × 1B float32 = 3 TB → memory tier 1B 已撞墙** |
| Multimodal disk-resident production case | 不涵盖 | **NEW: zero coverage despite CLIP 普遍存在 production** |

### CLIP embedding × disk tier storage 全表（NEW）

[per radford-2021-clip CLIP dim + wiki disk tier]

| Tier | 1B vectors storage | 10B | 100B | 1T |
|---|---|---|---|---|
| **CLIP 768-d float32** | 3 TB (memory hard) | 30 TB | 300 TB | 3 PB |
| **CLIP 768-d float16** | 1.5 TB (memory hard) | 15 TB | 150 TB | 1.5 PB |
| **CLIP 768-d int8 (SQ)** | 768 GB (memory soft) | 7.7 TB | 77 TB | 770 TB |
| **CLIP 768-d binary (1-bit per dim packed)** | 96 GB | 960 GB | 9.6 TB | 96 TB |

vs Traditional SIFT 128-d (wiki benchmark default):

| Tier | 1B SIFT 128-d | 10B | 100B | 1T |
|---|---|---|---|---|
| SIFT 128-d float32 | **512 GB** (memory soft) | 5 TB | 50 TB | 500 TB |
| SIFT 128-d int8 | 128 GB | 1.3 TB | 13 TB | 128 TB |

→ **Direct cost ratio**: CLIP 768-d 比 SIFT 128-d **多 6× storage**——multimodal production workload 在 memory tier 更快撞墙, disk tier 切换提前.

### CLIP 在各 disk tier 上的 production reality

[per wiki vendor support + CLIP embedding properties]

| Disk tier | CLIP-friendly? | wiki vendor support |
|---|---|---|
| Memory (HNSW) | **CLIP 768-d × ≤1B fit RAM 边界** | Milvus / Qdrant / Weaviate / Vespa / Pinecone all support |
| Memory + quantization | **CLIP 768-d × 1-10B fit RAM 边界 with int8/binary** | Qdrant BQ + Weaviate BQ + Vespa cell type |
| Local NVMe + SPANN-style | **CLIP 多 100B 可行 path** | Vespa SPANN |
| Local NVMe + DiskANN-graph | **CLIP 多 100B 可行 path** | Milvus DISKANN |
| Local disk + brute-force per-tenant | **CLIP multimodal per-user partition path** | Vespa Streaming Search (multimodal-aware?) |
| Object storage + SPFresh | **CLIP cost-sensitive 大规模 path** | Turbopuffer SPFresh |
| Distributed KV + single graph | **CLIP large-scale single-shard graph path** | DistributedANN (Bing-style, multimodal 不明示) |

### Multimodal disk-resident production frontier

[per wiki industry coverage]

- **HNSW memory + int8 CLIP**: 1B production case 普遍 (5 vendor 支持) 但无 case 公开实测
- **SPANN-style CLIP**: Vespa OSS path 理论支持 CLIP embedding, 但实际 production case 不公开
- **DiskANN CLIP**: Microsoft DiskANN 仓库技术上支持任意 cosine embedding, 但 multimodal production case 不公开
- **Turbopuffer SPFresh + CLIP**: docs 提及 "multi-modal data 应用" 但具体 scale 不公开
- **DistributedANN + CLIP multimodal**: Bing 50B vector 是否 multimodal 未明示

→ **Multimodal disk-resident large-scale production case 是 wiki 内 critical missing data point**.

### 决策表（updated 2026-05-11 post radford-2021-clip）

| Workload | 推荐方案 |
|---|---|
| **CLIP-style 1B + memory + cosine ANN** | HNSW (5 OSS vendor all support) |
| CLIP-style 1-10B + memory bound | + binary/int8 quantization (Qdrant BQ / Weaviate BQ / Vespa cell type) |
| CLIP-style 10-100B + cost-sensitive | **SPFresh + object storage (Turbopuffer)** |
| CLIP-style 10-100B + latency-priority | Vespa SPANN + 4-phase ranking (multimodal-aware tensor framework) |
| CLIP-style ≥100B + 单一 corpus + throughput | DistributedANN single-graph distributed (Bing-style) |
| Per-user multimodal RAG (≤1M docs/user) | Vespa Streaming Search + multi-tensor field |

### 已知盲区

- **CLIP 在各 vendor disk tier 上的 recall × latency × cost 实测**: 不公开
- **CLIP-specific quantization tolerance on disk tier**: binary CLIP + DiskANN/SPANN 实际 recall 退化曲线? wiki zero coverage
- **6-modality embedding (ImageBind 1024-d)**: 比 CLIP 768-d 多 33% storage, disk tier 切换更快撞墙——但 production case 完全空白
- **Multimodal-specific disk index optimization**: 是否 multimodal embedding 几何性质允许 disk layout 优化 (Starling-style)? wiki zero coverage

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
