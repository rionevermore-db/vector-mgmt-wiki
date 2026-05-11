---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/spann.md, systems/spfresh.md, systems/turbopuffer.md, systems/vespa.md, systems/milvus.md, systems/pinecone.md, concepts/lire.md, topics/disk-vs-memory-ann.md]
cited-count: 8
---

# Post-snapshot (turbopuffer-docs): giga-scale-sharding

## TL;DR (delta from vespa-docs post)

**Turbopuffer 引入 wiki 内最高 single-vendor 公开 production scale claim**——3.5T+ docs / 13PB+ total / 100B+ vectors queryable simultaneously / 100M+ namespaces——超过此前所有 wiki 内 vendor 数字（Vespa SPANN OSS / Pinecone 数字不公开 / Milvus 实测仅 100M / Microsoft Bing 1.5T 闭源）。**关键 NEW**：Turbopuffer 给出 16 节点 + 1TB RAM 假设的**完全不同 sharding 哲学**——不分 shard 而是分 **namespace**（10M-100M 个），每 namespace 独立 SPFresh + 独立 cache，节点 stateless 任意服务任意 namespace。这是 wiki 内**第一个 sharding 推到极致 fine-grained（≥10M namespace）的 production design**。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| SPFresh production case | 仅 Microsoft Bing implied (闭源) | **+ Turbopuffer commercial SaaS (frontier closed)** |
| 最高 vendor scale claim | Vespa SPANN OSS + Bing 1.5T (闭源) | **Turbopuffer 3.5T+ docs / 100B+ queryable (OSS-known)** |
| 16 节点千亿规模 sharding 路径 | (a) Vespa SPANN content cluster + group routing / (b) Vespa Streaming per-user | **+ (c) Turbopuffer namespace-fanout 100M+ per S3 prefix** |

### Turbopuffer namespace-fanout 拓扑（NEW）

[per sources/docs/turbopuffer/llms-full.txt §limits §architecture §multi-tenancy]

**16 节点 + 1TB RAM 假设下的 Turbopuffer 部署**：
- **不存在 "shard 千亿数据到 16 节点"**——这是 misframe
- Turbopuffer 哲学："3.5T+ docs 是 100M+ namespaces 的累积，每 namespace 独立"
- 16 节点 = 16 stateless query nodes + indexing nodes (可独立 autoscale)
- **每 namespace 独立 S3 prefix + 独立 SPFresh + 独立 cache**
- LB → query node (locality) → 该 node 服务该 namespace 的 query

```
3.5T docs ÷ 100M namespaces = avg 35K docs/namespace
                              ↓
                       每 namespace fits 1 GB
                              ↓
                  100M namespaces × 1 GB = 100 PB
                              ↓
                   all on S3 (object storage)
```

**性能假设**：
- Cold query: p50=343ms / p90=444ms 1M docs (NVMe miss, 3-4 roundtrip × 100ms)
- Warm query: p50=8ms 1M docs (NVMe hit)
- 16 节点 cache 容量 = 16 × NVMe (假设 4 TB) = 64 TB → 装 64M namespaces in NVMe
- LRU eviction → 老 namespace 被换出 → 用 [warm-cache API](sources/docs/turbopuffer/llms-full.txt §warm-cache) 预热 latency-sensitive namespace

### Turbopuffer vs Vespa 千亿规模 sharding 三路径对比

| | Vespa SPANN | Vespa Streaming | **Turbopuffer namespace-fanout** |
|---|---|---|---|
| Shard 粒度 | content cluster group + node | per-user partition | **100M+ S3-prefix namespace** |
| Compute 状态 | stateless container + stateful content | stateful content | **fully stateless query + indexing** |
| Cold latency 来源 | mem-miss → SSD | disk scan tenant partition | **S3 roundtrip × 100ms × 3-4** |
| Warm latency | < 10ms | < 10ms tenant brute-force | < 10ms NVMe hit |
| Failover | content node replica | content node | **任意 query node 立即接管 (HA = node count)** |
| 千亿规模 viable | ✓ via SPANN posting on-disk | ✓ via per-tenant <1M | ✓ via 100M+ namespaces |

### 千亿/万亿决策表（updated 2026-05-11 post turbopuffer-docs）

| Workload | 推荐方案 |
|---|---|
| **多租户极致 (1M+ tenant 独立 data)** | **Turbopuffer namespace-as-tenant** |
| **想要 0 ops + 闭源 SaaS + 千亿规模** | **Turbopuffer (OSS-known production scale ceiling)** |
| Multi-tenant + per-tenant 数据小 (personal AI) | Vespa Streaming Search |
| Shared corpus 千亿 + 复杂 ranking + ML rerank | Vespa SPANN + 4-phase ranking |
| 千亿 + 简洁 OSS Rust | Qdrant + DiskANN-style (docs 不详) |
| 千亿 + 多 index_type | Milvus DISKANN / SPARSE_INVERTED |
| Billion-scale + 闭源 SaaS managed | Pinecone slab |

### Turbopuffer 16 节点 + 1TB RAM 假设的实测 P99

**问题原文假设**: 768-dim, 16×128U+1TB nodes, P99 < 50ms, weekly full refresh

**Turbopuffer 路径分析**:
- Storage: 1T docs × 768 × 2 (f16) = 1.5 TB / namespace 平均 ≈ 15 KB/namespace 100M ns → 1.5 PB 总——全 S3
- Compute: 16 nodes × 4TB NVMe = 64 TB cache → fit 64M × 1MB namespace
- Warm hit ratio ~64%（若 LRU + Zipf 分布则更高）
- P99 警报：cold query 仍 343ms-444ms——需 namespace pinning 或 warm-cache hint for **fully warm** 路径
- Weekly full refresh: 1T docs / (10K writes/s/namespace × 100M ns) → 极快
- 实际 production: **每 namespace 1K+ QPS**——只要每 namespace 不超 1K QPS 即可保证 P99 < 50ms（warm）

### 已知盲区

- **Turbopuffer 100M namespace × 35K doc 平均的实际客户案例**：B2B SaaS 类型 specifics 不公开
- **Single namespace 跑 1B docs (超 500M ceiling) 的 sharding**：必须 id % N 手动 shard，Turbopuffer 不自动
- **Cold p99 在 100M namespace 集群下**：单 LRU pool 还是 per-org partition？docs 不细谈
- **Multi-region routing**：跨 region 写 / 读 一致性 docs 提 cross-region backup, 但 active-active 不暴露
- **16 节点假设的 Vespa SPANN vs Turbopuffer head-to-head**：未公开

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/lire.md](../../concepts/lire.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
