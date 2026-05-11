---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/spann.md, systems/spfresh.md, systems/turbopuffer.md, systems/vespa.md, systems/milvus.md, concepts/hnsw.md, topics/disk-vs-memory-ann.md]
cited-count: 7
---

# Post-snapshot (turbopuffer-docs): scale-tier-shifts

## TL;DR (delta from vespa-docs post)

**Turbopuffer 把 wiki 内 tier-shift 表格的右端推到新极端**：3.5T+ docs / 13PB+ total / 100M+ namespaces 是**wiki 内 OSS-known 最高 vendor scale claim**。**关键 NEW**：Tier 5 (≥1T) 首次有 OSS-known production case——之前仅 Microsoft Bing 1.5T (闭源 SPANN)，现在 Turbopuffer commercial SaaS 是**第二个 ≥1T 数据点**。**另一关键 NEW**：Tier-shift 在 Turbopuffer 体系下有**完全不同的形状**——不是 "regime shift across tiers"，而是 **"fixed architecture works at all tiers via namespace fanout"**。Object storage primary + 100M+ namespace 让 1B 和 1T 用同一个 deployment shape。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| SPFresh production case | 仅 Bing implied | **+ Turbopuffer (OSS-known frontier closed)** |
| Tier 5 (≥1T) production data point | Bing 1.5T 闭源 only | **+ Turbopuffer 3.5T+ (OSS-known)** |
| Object storage tier (object storage as primary) | wiki 未列 | **+ Turbopuffer "all tiers same architecture"** |
| Per-tenant brute-force tier (Vespa Streaming) | Vespa only | **不变** |

### 标准 tier-shift 表（updated 2026-05-11 post turbopuffer-docs）

| 规模 | 索引选择质变点 | 存储介质质变点 | 并行能力质变点 |
|---|---|---|---|
| ≤ 10 亿 (≤ 1B) | HNSW / IVF + memory all-in | 全内存 | 单机或 2-3 节点 |
| 10 亿 - 百亿 (1B-10B) | HNSW + quantization | 内存边界 临近 SSD | 多节点分 shard |
| 百亿 - 千亿 (10B-100B) | **SPANN/SPFresh tier shift** — centroids in-mem / posting on-disk | **必须 SSD/NVMe**（Vespa SPANN, Milvus DISKANN, etc.）OR **Object storage**（Turbopuffer SPFresh） | 路由层成必需 |
| 千亿 - 万亿 (100B-1T) | SPANN production (Vespa OSS) **OR SPFresh production (Turbopuffer OSS-known)** | NVMe + 高 IOPS / 分布式 KV (Pinecone slab) OR **S3 (Turbopuffer)** | 复杂 fanout + result merge |
| **≥ 1T** | Bing 1.5T (SPANN 闭源) + **Turbopuffer 3.5T+ (SPFresh OSS-known)** | NVMe 闭源 (Bing) OR **S3 OSS-known (Turbopuffer)** | namespace-fanout (100M+ S3 prefix, Turbopuffer) |
| Special: per-tenant brute-force | Vespa Streaming (no index) | per-tenant disk partition (45 B/doc) | 单 node billions/doc |
| Special: per-tenant namespace fanout | **Turbopuffer (SPFresh per namespace)** | per-namespace S3 prefix (100M+) | **任意 query node 服务任意 namespace** |

### Turbopuffer 引入的"unified architecture across tiers"（NEW）

[per sources/docs/turbopuffer/llms-full.txt §architecture §limits]

之前 tier-shift 隐含假设：随 scale 增长**架构必须变形**（HNSW → SPANN → Pinecone slab）。Turbopuffer 反假设：
- 1B docs in 1 namespace + 100M namespaces × small docs = same architecture
- 物理上：同一 query node + same NVMe cache + same S3 protocol
- Tier-shift 仅在**单 namespace 内部**: 1 namespace > 500M docs/2TB 必须 manual shard

→ Turbopuffer 把传统 tier-shift 从"vendor / system 级"降到"namespace 级"——同 vendor 可同时跑 small (KB/doc) namespace + medium (MB/doc) namespace + 500M-doc large namespace.

### Tier 5 (≥1T) production 真正实证（NEW）

之前 wiki 内 ≥1T 仅 Bing 1.5T (SPANN 闭源)——单点闭源 ground truth, OSS 不可验证。

Turbopuffer 公开声明 3.5T+ docs production——**OSS-known production case**:
- 不是 OSS 软件 (Turbopuffer 闭源 SaaS), 但**数字公开 + 客户可独立 sign-up 验证**
- 任何客户可创建 namespace + 发数据 → 看到自己数据在 production cluster 内运行
- 比 Bing 数字（仅论文 + 内部确认）更可独立验证

→ Tier 5 frontier 从"single internal data point" 升级为"two independent production data points"。

### Tier-shift 驱动 vs Turbopuffer "no shift" 哲学

传统 tier-shift 驱动：
1. **内存撞墙** → 加 quantization 或 切 SSD
2. **SSD 撞墙** → 切 object storage（极少 production）
3. **单 graph 太大** → 切 routed (SPANN / segment)

Turbopuffer **倒置驱动**：
1. **从第一天就 object storage primary** → 不存在"撞墙 → 切 object storage"
2. **从第一天就 namespace partition** → 不存在"single graph 太大"
3. **从第一天就 SPFresh + LIRE** → 不存在"static → streaming rebuild"

→ Trade-off: Turbopuffer cold p50=343ms 是永久存在的 baseline——peer DBMS 1B in-RAM HNSW 单 query 1ms 内仍是 Turbopuffer 不可达的 floor。**Turbopuffer 不是 dominant for all workloads**——是 dominant for cost-sensitive + multi-tenant + cold-latency-tolerant。

### 已知盲区

- **Turbopuffer 100M namespace × small data 实际 workload composition**：B2B SaaS / personal AI / RAG which dominates 未明示
- **Tier 5+ (1T+) Turbopuffer 单 query 实测 P99**：cold p99 在 100B+ queryable scale unknown
- **DiskANN production 主流案例**: 仍主要学术 + Milvus
- **Tier 4 万亿 (1T+) 多 vendor 实证**: 仅 Bing + Turbopuffer (2 数据点)

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
