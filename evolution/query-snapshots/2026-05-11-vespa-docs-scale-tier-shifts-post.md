---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [systems/spann.md, systems/diskann.md, systems/vespa.md, systems/milvus.md, concepts/hnsw.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (vespa-docs): scale-tier-shifts

## TL;DR (delta from weaviate-docs post)

**Vespa 引入一个 wiki 内全新 tier 概念**："per-user / per-tenant partition size" tier——当 query 只触及一个 user 的数据，**Streaming Search (no index, brute-force) 在 billion docs/node 仍然成立**，是 [SPANN](../../systems/spann.md) / [DiskANN](../../systems/diskann.md) 之外的**反方向 tier shift**："索引开销 vs partition 实际大小"成为新质变点。**关键 NEW**：Vespa 也是**第二个 SPANN production case**，确认"≥百亿即转 SPANN-style centroid-on-mem + posting-on-disk"作为 production tier shift，**不再仅 Microsoft Bing 单点**。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| SPANN ≥百亿 tier shift 实证 | Microsoft Bing only (闭源) | **+ Vespa OSS (2nd) — production frontier closed** |
| 反方向 "no-index brute-force" tier | wiki 未提 | **+ Vespa Streaming Search (45 B/doc, billions/node)** |
| Tier shift 标准 | 内存 → SSD; HNSW → SPANN/DiskANN; 单图 → 路由 | **+ per-tenant partition size 判定"要不要建 index"** |

### 标准 tier-shift 表（updated 2026-05-11 post vespa-docs）

| 规模 | 索引选择质变点 | 存储介质质变点 | 并行能力质变点 |
|---|---|---|---|
| **≤ 10 亿** (≤ 1B) | HNSW / IVF + memory all-in | 全内存可控 (1TB RAM 装得下 768-dim × 1B × 4B = 3 TB) | 单机或 2-3 节点 |
| **10 亿 - 百亿** (1B - 10B) | HNSW + quantization (PQ/RQ/binary) 还能 in-memory | 内存边界临近, SSD 开始 attractive (DiskANN) | 多节点分 shard |
| **百亿 - 千亿** (10B - 100B) | **SPANN tier shift** — 内存放 centroids, disk 放 posting lists; 或 DiskANN; HNSW pure-memory 不再现实 | **必须 SSD/NVMe** — RAM 仅装 centroids/cache | **路由层成必需** (centroid router + shard query fanout) |
| **千亿 - 万亿** (100B - 1T) | SPANN production deployment 唯一实证 path (Microsoft Bing + **Vespa OSS**) | NVMe + 高 IOPS 储池 + 分布式 KV (Pinecone slab) | 复杂 fanout + result merge; coordinator HA |
| **特殊：per-tenant brute-force tier** | **Vespa Streaming Search (no index!)** — 当 query 仅扫 per-user partition (<1M docs) | 完全 disk-scan, RAM 仅 cache | 单 node billions/doc 可承载 |

### Vespa Streaming Search 引入的新维度（NEW）

[per sources/docs/vespa/llms-full.txt §Streaming Search]

之前 tier-shift 全部围绕"corpus 大、单 query 必须 ANN 缩小搜索空间"假设。Vespa Streaming Search 暴露**反假设**：
- 当 corpus 是 multi-tenant，且 query 仅查 1 个 tenant 的数据
- 该 tenant 的 partition 可能只是 100K-1M docs
- 单 query brute-force 扫该 partition 仅几 ms
- **建 ANN index 反而是 overhead**（构建开销 + 内存开销 + 更新开销 都是浪费）

→ 这是 wiki 内**首个把"per-tenant data size 是否值得建 index" 列入 tier shift criteria 的系统**。Personal AI assistant / RAG over personal data / 企业内每用户 namespace 都是这场景。

### SPANN 第二个 production 实证（NEW）

Microsoft Bing 是闭源 SPANN production（论文 [Chen 2021](../../systems/spann.md)）；Vespa OSS 实现 SPANN——**任何人都可下载 Apache-2.0 OSS 验证**。SPANN production frontier 不再单点。

### Tier-shift 决策驱动（updated 2026-05-11 post vespa-docs）

1. **Tier 1 (≤1B)**: HNSW + memory 是默认正确解
2. **Tier 2 (1-10B)**: 加 quantization 拖延"内存撞墙"
3. **Tier 3 (10-100B)**: 决定是否切 SPANN / DiskANN —— 取决于 hardware (SSD 可用 + IOPS 富余) 和 update frequency (SPFresh 配套)
4. **Tier 4 (≥100B)**: SPANN production 唯一 OSS-validated path (Vespa); 否则闭源 SaaS path (Pinecone slab)
5. **Special tier**: 多租户 + per-tenant 小 → **Vespa Streaming Search 跳过整个 ANN tier ladder**

### 已知盲区

- **Vespa SPANN 实测 100B+ latency**：docs 提机制无实测数字
- **Streaming Search per-tenant threshold**: per-tenant docs 上限到多大 brute-force 失效，docs 未量化
- **DiskANN production 案例**: 仍主要学术；Milvus DISKANN production benchmark 公开数据少
- **Tier 4 万亿 (1T+) 实证**: Microsoft Bing 1.5T 是唯一公开数字, Vespa 数字 unknown

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
