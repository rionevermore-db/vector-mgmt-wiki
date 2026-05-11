---
title: Turbopuffer（object-storage-native 闭源 commercial SaaS）
type: system
sources: [turbopuffer-docs, xu-2023-spfresh, chen-2021-spann]
related: [pinecone.md, milvus.md, qdrant.md, weaviate.md, vespa.md, spfresh.md, spann.md, diskann.md, ../concepts/lire.md, ../concepts/product-quantization.md, ../topics/disk-vs-memory-ann.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/attribute-filtering.md, ../topics/index-selection.md]
created: 2026-05-11
updated: 2026-05-11
---

# Turbopuffer

**TL;DR**: Turbopuffer Inc. 闭源 commercial SaaS（Rust 实现），**wiki 内第二个 closed SaaS**（继 [Pinecone](./pinecone.md) 之后，两种闭源哲学对比）。核心独特性：(1) **object-storage-native 架构**——S3/GCS 是唯一 stateful 依赖，所有 compute 完全 stateless（query nodes + indexing nodes 双 compute-compute 分离 + autoscale）；(2) **SPFresh 首个 OSS-known production deployment**——[SPFresh](./spfresh.md) (SOSP 2023, Microsoft Research) 学术系统 2026 进入 commercial production（**SPFresh frontier 关闭**——之前 wiki 内 SPFresh 仅论文 + Bing 内部 implied）；(3) **LSM tree natively on object storage**——非 local-disk LSM 移植，专为 object storage roundtrip 经济性设计；(4) **3-tier cache hierarchy**（Memory + NVMe SSD + Object Storage）+ 智能缓存；(5) **Strong consistency by default**——WAL on object storage 提供，~10ms latency floor 由 object storage `GET IF-NOT-MATCH` 决定；(6) **100M+ namespace 一等公民**——每 namespace 是 object storage prefix，无架构约束，**multi-tenancy 默认设计点**而非追加功能。生产观察：3.5T+ docs / 13PB+ total / 100B+ vectors queryable simultaneously / 100M+ namespaces / 10M+ writes/s @ 32GB/s / 25k+ QPS / 90-100% recall@10。20 个公开 region (AWS + GCP) + BYOC (AWS/GCP/Azure)。**No OSS, no free tier**。[per sources/docs/turbopuffer/llms-full.txt §architecture §limits §tradeoffs]

## 与 wiki 内其他 system 的定位差异

| | Pinecone | Milvus | Qdrant | Weaviate | Vespa | **Turbopuffer** |
|---|---|---|---|---|---|---|
| OSS / closed | **闭源 SaaS** | OSS Go | OSS Rust | OSS Go | OSS C++/Java | **闭源 SaaS** |
| Storage 主层 | NVMe local + adaptive | NVMe + S3 (segment) | NVMe local | NVMe local | NVMe local | **Object storage (S3) primary** |
| Compute 状态 | 闭源（不公开） | streaming/query/data node | 单 binary stateful | 单 binary stateful | container stateless + content stateful | **全 stateless (query + indexing)** |
| ANN 算法 | 不公开 (slab) | HNSW/IVF*/DISKANN/CAGRA/SPARSE | HNSW only | HNSW only | HNSW + SPANN + Streaming | **SPFresh (centroid-based)** |
| Multi-tenancy 路径 | 不公开 | partition | namespace | named vectors / multi-tenant collections | content cluster groups | **namespace = S3 prefix（100M+ 一等公民）** |
| Strong consistency | 不明示 | tunable | tunable | tunable | tunable | **default; ~10ms floor** |
| Min latency (warm) | p99 ~10ms (闭源) | 数 ms | 数 ms | 数 ms | 数 ms | **p50=8ms 1M docs** |
| Cold latency 哲学 | 不暴露 | 不暴露 | 不存在（hot only） | 不存在 | 不存在 | **p50=343ms / p90=444ms 1M docs，explicit** |
| Vendor 哲学 | "managed simplicity" | "tool box" | "do HNSW well" | "AI-native primary DB + agent stack" | "search engine + tensor pipeline" | **"object-storage native + cheap massive scale"** |

**核心论点**：Turbopuffer 是 wiki 内**第一个把"object storage = primary storage"作为 first principle 设计的 production system**。peer DBMS（Milvus segment 模型 / SPANN / DiskANN）把 SSD/object storage 当作"内存装不下时的降级路径"；Turbopuffer **从架构第一天就假设所有数据都在 object storage**，compute 是无状态 cache 层。这种倒置带来 cost reduction（"10× cheaper than peer" 的来源），代价是 cold query 必然存在（p50=343ms / p90=444ms 1M docs）。

## 架构图

```
                        ╔═ turbopuffer ════════════════════════════╗
╔════════════╗          ║                                          ║░
║            ║░         ║  ┏━━━━━━━━━━━━━━━┓     ┏━━━━━━━━━━━━━━┓  ║░
║   client   ║░───API──▶║  ┃    Memory/    ┃────▶┃    Object    ┃  ║░
║            ║░         ║  ┃   SSD Cache   ┃     ┃ Storage (S3) ┃  ║░
╚════════════╝░         ║  ┗━━━━━━━━━━━━━━━┛     ┗━━━━━━━━━━━━━━┛  ║░
                        ╚══════════════════════════════════════════╝░
```

```
                   ╔═══turbopuffer region═════════════╗
                   ║      ┌─────────────────────────┐ ╠──┐
                   ║      │     ./tpuf indexer      │ ║░ │   ╔═══Object Storage══════════════╗
                   ║      └─────────────────────────┘ ║░ │   ║ ┏━━Indexing Queue━━━━━━━━━━━┓ ║░
                   ║      ┌─────────────────────────┐ ║░ │   ║ ┃■■■■■■■■■                  ┃ ║░
                   ║      │      ./tpuf query       │ ║░ │   ║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━┛ ║░
                   ║      │┌─Memory Cache──────────┐│ ║░ │   ║ ┏━/{org_id}/{namespace}━━━━━┓ ║░
                   ║      ││■■■■■■■■■■             ││ ║░ └──▶║ ┃ ┏━/wal━━━━━━━━━━━━━━━━━━┓ ┃ ║░
                ┌──╩─┐    │└───────────────────────┘│ ║░     ║ ┃ ┃■■■■■■■■■■■■■■■◈◈◈◈    ┃ ┃ ║░
╔══════════╗    │ LB │───▶│┌─NVMe Cache────────────┐│ ║░     ║ ┃ ┗━━━━━━━━━━━━━━━━━━━━━━━┛ ┃ ║░
║  Client  ║───▶│    │    ││■■■■■■■■■■■■■■■■■■■■■  ││ ║░     ║ ┃ ┏━/index━━━━━━━━━━━━━━━━┓ ┃ ║░
╚══════════╝░   └────┘    │└───────────────────────┘│ ║░     ║ ┃ ┃■■■■■■■■■■■■■■■        ┃ ┃ ║░
                          └─────────────────────────┘ ║░     ║ ┃ ┗━━━━━━━━━━━━━━━━━━━━━━━┛ ┃ ║░
                   ╚══════════════════════════════════╝░     ║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━┛ ║░
                                                              ╚═══════════════════════════════╝░
```

[per sources/docs/turbopuffer/llms-full.txt §architecture]

3 类组件、3 种存储层：
- **Query nodes (`./tpuf query`)**：handle API reads/writes，stateless，本地 NVMe SSD + Memory cache
- **Indexing nodes (`./tpuf indexer`)**：异步建索引（compute-compute separation——不与 query 共享 CPU/RAM）
- **Object Storage**：每 namespace 占 `s3://tpuf/{org_id}/{namespace}/` prefix，下属 `/wal` + `/index` 两个目录

3 种存储层（从热到冷）：
- **Memory cache** — frequently accessed namespaces
- **NVMe SSD cache** — recently queried namespaces（cold query 后自动填充）
- **Object Storage (S3/GCS)** — durable source of truth；**唯一 stateful 依赖**

[per sources/docs/turbopuffer/llms-full.txt §concepts §guarantees]

## 数据流 / 控制流

### 写入路径

1. Client → LB → Query node
2. Query node 把 batch 内 writes append 到 namespace 的 WAL：`s3://tpuf/{ns}/wal/00X`
3. 当 object storage 确认写成功 → query node return 200 OK（**writes durable upon return**）
4. 异步：Indexing node 读 WAL 新条目 → 建 index → 写 `s3://tpuf/{ns}/index/...`
5. Query node 通过 object storage metadata 发现新 index file → 缓存到 NVMe

[per §architecture §wal]

**关键约束**：每 namespace 1 WAL entry/秒（group commit）。并发 writes 自动 batch。p50=200ms 写延迟，但吞吐 10K+ vectors/sec/namespace；全集群 10M+ writes/s @ 32 GB/s 已观察。

### 查询路径（cold / warm 两阶段）

**Cold query (第一次访问 namespace)**：
1. LB → query node（任意 node 都可服务任意 namespace）
2. Query node 从 object storage 读 metadata（roundtrip 1, ~100ms）
3. 读 centroid index（roundtrip 2, ~100ms）
4. 定位最近 centroids → fetch posting lists（roundtrip 3, ~100ms）
5. 计算距离、返回 top-K + 缓存到 NVMe
- **p50=343ms / p90=444ms for 1M documents** [per §architecture]

**Warm query (cache hit)**：
1. LB → 同 query node (locality)
2. NVMe/Memory cache 命中 → 仅一次 object storage roundtrip 做 strong consistency check
- **p50=8ms / p99 数十 ms for 1M documents**

[per §architecture §index]

### Strong consistency 默认 + 10ms latency floor

[per §guarantees §tradeoffs]

每次 consistent query 都对 object storage 做 `GET IF-NOT-MATCH` 检查 namespace 元数据版本。这给：
- **S3 metadata p50=10ms / p90=17ms**
- **GCS metadata p50=12-18ms / p90=15-25ms (region-dependent)**

→ **Strong consistency 的 10ms floor 是 object storage 决定的，不是 Turbopuffer 限制**。Application 想 sub-10ms 必须切 eventual consistency（最多 1 小时 staleness, 实测 99.8% query 仍 consistent）。

## 关键设计决策

### 1. Object storage as primary, not fallback

[per §architecture §concepts]

**Trade-off**：放弃 sub-10ms cold latency / 50ms-级别 write latency → 换 (a) 极度便宜（"10× cheaper" claim）, (b) 极度可扩展（millions of namespaces, trillions of docs）, (c) 极度简单的 HA（"any query node serves any namespace"——没有 sharding 状态管理）。

vs peer DBMS：
- **Milvus segment** + S3：S3 是 backup/long-term, hot data 在 SSD
- **SPANN**：centroids 在内存, posting lists 在 SSD（不是 object storage）
- **DiskANN**：graph 在内存, vectors 在 SSD
- **Vespa**：全部 local NVMe；Streaming Search 是 disk scan but not object storage
- **Turbopuffer**：**全部在 object storage, cache 是优化**

### 2. SPFresh as ANN index（frontier closure）

[per §architecture §concepts §vector]

> Vector indexes are based on [SPFresh](https://dl.acm.org/doi/10.1145/3600006.3613166). SPFresh is a centroid-based approximate nearest neighbour index... A centroid-based index works well for object storage as it minimizes roundtrips and write-amplification, compared to graph-based indexes like HNSW or DiskANN.

→ **Wiki 内 [SPFresh](./spfresh.md) 首个 OSS-known production deployment**。SPFresh paper [xu-2023-spfresh] (Microsoft Research, SOSP 2023) 之前在 wiki 是"研究系统 + 论文 author 来自 Bing/SPANN 团队 implied 内部 production"——Turbopuffer 2026 公开 production deployment **彻底关闭 SPFresh frontier**（学术 → commercial production 仅 3 年）。

**为什么 centroid-based 适合 object storage**（Turbopuffer 论点）：
- Graph-based (HNSW/DiskANN): 单 query 需要 traverse ~log(N) 节点 → 高 roundtrip 数 × ~100ms = cold query 慢
- Centroid-based (SPFresh/SPANN-family): 单 query 仅 (1) 读 centroid index → (2) 一次 batch fetch posting lists → low roundtrip
- **SPFresh 比 SPANN 优势**：incremental in-place update（[LIRE](../concepts/lire.md) protocol）——不需要周期 rebuild centroids

### 3. Compute-Compute separation（双 stateless 节点池）

[per §architecture §concepts §guarantees]

**Query nodes** + **Indexing nodes** 两个独立 autoscale pool：
- Query node CPU 不会被 indexing 抢
- Indexing node 写完后 query node 通过 object storage 同步发现
- Both pools 任意 node fail → 另一 node 立即接管（HA = node count）

→ Vespa 也有 stateless container / stateful content cluster 分离，但 Vespa **content cluster 是 stateful**；Turbopuffer **all compute is stateless**，HA 不需要 replica 协调，因为 object storage 是唯一权威。

### 4. WAL natively on object storage

[per §architecture §wal]

`s3://tpuf/{ns}/wal/{seq_number}` 是 namespace 的 write-ahead log。每文件是一个 batch。这是 **wiki 内首个"WAL 直接 on object storage"系统**——peer DBMS 的 WAL 都是 local disk 或 Kafka/Pulsar。

**含义**：
- Writes return 200 OK 即 durable（object storage replication 11-9s）
- 无需独立 consensus/Raft——object storage 提供 atomic conditional writes
- 任意 query node 可读任意 WAL → stateless

### 5. LSM tree natively on object storage

[per §concepts]

> Most LSM trees are built for local disk. turbopuffer's is built natively on object storage.

**关键差异 vs 传统 LSM**：
- 传统 LSM (RocksDB / LevelDB): sequential write 优化 local disk seek
- Turbopuffer LSM: optimized for **minimizing object storage roundtrips per query**
- Compaction = indexing node merge sorted runs，结果写回 object storage

→ Pure single-system view, Turbopuffer 是 **wiki 内首个 "object storage LSM" production system**。

### 6. Namespace as architectural primitive（100M+ 一等公民）

[per §concepts §multi-tenancy §limits]

每 namespace 是 object storage prefix：`s3://tpuf/{org_id}/{namespace}/`。架构上：
- **Unlimited namespaces** (Limits 表：100M+ observed)
- 每 namespace 独立 schema / vector index / FTS index / attribute indexes
- Multi-tenant SaaS 推荐"一 tenant 一 namespace"——避免 filter overhead

→ vs Weaviate "collection-level multi-tenancy"（collection within class）或 Pinecone "namespaces within index"——Turbopuffer 是**最 aggressive 的 namespace-as-tenant** 模型。

**Performance 建议**："Smaller namespaces will be faster to query and index"——不要 monolithic namespace + filter，应 split by natural data boundary。

### 7. Strong consistency by default (10ms floor)

[per §guarantees §concepts]

Wiki 内首个**默认 strong consistency** 的 production vector DBMS。peer DBMS 默认都是 eventual。
- Strong: every query 验证 object storage metadata → 看到所有 prior writes
- Eventual: 仅扫 128 MiB unindexed data buffer，最多 1 hour stale（worst case，99.8% 仍 consistent）

trade-off 明确：strong cost ~10ms latency floor; eventual buys sub-10ms。Application 自选。

### 8. Pinning (April 2026)

[per §pinning §pricing-log]

为高 QPS namespace 预留 compute + cache：
- **Multi-tenant (default)**: 共享 compute pool, per-query TB Queried 计费
- **Pinned**: 预留 query node + NVMe，billed in GB-hours
- Break-even: ~10 QPS sustained
- 解决 noisy neighbor + cold query 的高 QPS 用户痛点

## Scale 边界

[per sources/docs/turbopuffer/llms-full.txt §limits]

**Production observed**:

| Metric | Production observed | Current product limit |
|---|---|---|
| Max docs global | **3.5T+ @ 13PB+** | Unlimited |
| Max docs queryable simultaneously | **100B+ @ 10TB** | Unlimited (manual shard via namespace) |
| Max docs per namespace | 500M+ @ 2TB | 500M @ 2TB |
| Max namespaces | **100M+** | Unlimited |
| Max write throughput global | 10M+ writes/s @ 32GB/s | Unlimited |
| Max write throughput per namespace | 32K+ writes/s @ 64MB/s | 10K writes/s @ 32MB/s |
| Max queries global | 25K+ QPS | Unlimited |
| Max queries per namespace | 1K+ QPS | 1K+ QPS (read replicas can scale) |
| Vector dim (dense) | — | 10,752 |
| Vector recall@10 | 90-100% | 90-100% auto-tuned |

→ **3.5T docs / 100M namespaces** 是 wiki 内**单 vendor 最高 production scale claim**：
- vs Pinecone (billion-scale claimed, 数字不公开)
- vs Milvus (Manu paper 实测到 100M; Zilliz cloud 数字不公开)
- vs Vespa (Microsoft Bing SPANN ~1.5T 是 Vespa-blueprint, 但非 Vespa 自家 prod)
- vs Microsoft Bing (1.5T+ vectors, 闭源 production)

### 瓶颈

- **Per-namespace 2TB ceiling**（500M docs at 4KB avg）——超 2TB 必须 manual id % N shard 到多 namespace
- **Per-namespace 1K+ QPS**——可加 read replicas (max 3 observed; 未来 auto-scale)
- **128 MiB unindexed buffer**——eventual consistency search 上限
- **2 GB ingest 队列**——indexing 落后超此值，writes return HTTP 429（背压）
- **Cold query 不可消除**——只能 pre-warm 或 pinning

## 生产案例

[per sources/docs/turbopuffer/llms-full.txt + 公开材料]

Turbopuffer 不公开客户清单。公开提及的客户使用案例 (per docs / blog signals)：
- "100B+ vectors @ 10TB" 的 ann-v3 blog 案例——某客户 production
- B2B SaaS multi-tenant 应用（"each tenant's data is isolated in its own namespace"——target market）
- AI Assistant / RAG / Chat 应用 per-user namespace 隔离（Streaming Search 哲学的 commercial 对应物，但 Turbopuffer 是 SPFresh-based ANN，不是 brute-force）

**已知 customer signals**（公开 blog/social）：Cursor, Notion, Linear 等 AI-native SaaS 工具——但未在 docs 明示。

## Open Questions

- **SPFresh implementation 细节**：Turbopuffer 闭源 Rust 实现 SPFresh——可能用 [LIRE](../concepts/lire.md) 协议但 [xu-2023-spfresh] 论文是 C++ + SPDK；Rust port + object storage adaptation 必然非简单 fork。具体 fork 程度 docs 不公开
- **SPFresh vs object storage roundtrip cost**：原论文假设 raw SSD (SPDK)；object storage 是高 latency 高 throughput。SPFresh centroid + posting list 适配 object storage 时是否仍保留 LIRE NPA 性质？未明示
- **Cold query 加速 path**：cold p50=343ms / p90=444ms 还有空间吗？Warm cache hint API 是工程层 work-around，algorithm 层是否有降 roundtrip 的研究空间？
- **3.5T docs 实际部署细节**：100M namespace 平均 35K docs/namespace——是哪种 workload？per-user RAG / SaaS B2B / multi-tenant 嵌入？
- **Recall@10 90-100% auto-tuning**：什么算法？是 SPFresh centroid 数自动调还是 nprobe 调？docs 未明示
- **Turbopuffer vs [Pinecone](./pinecone.md) 同代闭源 SaaS**：两者都是 closed SaaS only，但**透明度谱系**两端：Pinecone 完全不公开算法（slab adaptive 黑盒）；Turbopuffer 明示 SPFresh + LSM + WAL on object storage 全 stack——transparency 反差是 wiki 内最显眼对比
- **Turbopuffer vs [Vespa Streaming Search](./vespa.md)**：两者都"对 per-user / per-tenant 小数据高效"但走不同路径——Vespa Streaming 是 brute-force no-index 45 B/doc；Turbopuffer 是 SPFresh + namespace-per-tenant。**多大 per-tenant data 之上 Turbopuffer 比 Vespa Streaming 经济**？wiki zero coverage
- **Object storage 之外的 storage 候选**：Turbopuffer 哲学是 object storage native——但 R2 / B2 / MinIO / Azure Blob 等其他 object storage 实测是否同等 viable？BYOC 路径技术上支持但 docs 不细谈非 S3/GCS
- **OSS prospect**：tradeoffs 表明示"For the current phase... commercial-only model to maintain... rapid development. While we don't offer a free tier or open source version, you can run turbopuffer in your own cloud (BYOC)"——unlike Pinecone 的"永远闭源"姿态，Turbopuffer 措辞留 OSS 余地
- **Continuous recall monitoring 算法**：1% live query traffic 自动 sample → run exhaustive search compare——具体实现 algorithm docs 未深入；与 [recall endpoint](sources/docs/turbopuffer/llms-full.txt §recall) 接口的语义关系

Cited by: 待 query 引用
