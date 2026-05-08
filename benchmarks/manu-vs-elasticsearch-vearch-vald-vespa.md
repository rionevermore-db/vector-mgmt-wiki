---
title: Manu vs Elasticsearch / Vearch / Vald / Vespa (SIFT10M / DEEP10M)
type: benchmark
sources: [guo-2022-manu]
related: [../systems/milvus.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../benchmarks/milvus-vs-prior-sift10m-deep10m.md]
created: 2026-05-08
updated: 2026-05-08
---

# Manu vs Elasticsearch / Vearch / Vald / Vespa

**TL;DR**: Manu (= [Milvus 2.x 学术论文](../systems/milvus.md)) §5 的核心实验。在 SIFT10M (L2) / DEEP10M (内积) 上 Manu **HNSW 与 IVF_FLAT 双双系统性击败**全部四个开源 vector engine baselines。Manu 比 Elasticsearch 高 **数倍** throughput；比 Vearch 高 **数倍**；与 Vald (NGT) / Vespa (HNSW) 同算法但 Manu 实现 throughput 显著更高。还含 24h elasticity 实测（动态扩缩 query nodes 0.5×-2×）+ scalability 实测（query nodes 2-10 线性 / dataset 20M-100M 倒数）。[guo-2022-manu §5]

## 实验设置

[guo-2022-manu §5.2]

- **平台**：AWS EC2 m5.4xlarge（每 worker），Amazon Linux AMI 5.4.129
- **集群配置**：默认 2 query nodes + 1 data node + 1 index node
- **数据集**：
  - **SIFT (Euclidean)**：128-d byte vectors
  - **DEEP (Inner Product)**：96-d byte vectors
  - 取 sub-datasets（10M 或 100M 取决于实验）
- **对手**（**所有 vector engine baseline**，无 Faiss / SPANN / DiskANN 直接对比）：
  - **Elasticsearch 8.0** with vector search plugin（disk-based）
  - **Vearch**（Faiss + 三层 broker-searcher-blender 聚合）
  - **Vald**（仅 NGT 支持）
  - **Vespa**（仅 HNSW 支持）
- **Manu 索引**：IVF-FLAT (m=4096) + HNSW (M=16, ef=200)
- **Query**：top-50；recall 阈值 ≥ 0.8

## 主结果（SIFT10M / DEEP10M, [Fig 8]）

### Recall vs Throughput 曲线

定性顺位（基于 [Fig 8] 描述）：

| 排名 | Engine + Index | SIFT10M throughput @ 0.95 recall（推断） |
|---|---|---|
| 1 | **Manu HNSW** | **>8000 QPS** |
| 2 | **Manu IVF-FLAT** | **>4000 QPS** |
| 3 | Vald NGT | ~2000 QPS |
| 4 | Vespa HNSW | ~1000-2000 QPS（Vespa 同算法但慢得多） |
| 5 | Vearch HNSW | ~500-1000 QPS |
| 6 | Vearch IVF-FLAT | ~500 QPS |
| 7 | Elasticsearch HNSW | <500 QPS |

[guo-2022-manu §5.2 第 5 段] 论文论证：
- **Elasticsearch 慢因为 disk-based**——所有数据 SSD（不缓存到 memory）
- **Vearch 慢因为三层 broker 聚合**（broker → searcher → blender）开销
- **Vald / Vespa 同算法但 Manu 实现更优**——论文归因于"better implementations with optimizations for CPU cache and SIMD"

> **wiki 解读**：与 [Milvus 1.x 论文](./milvus-vs-prior-sift10m-deep10m.md) 中 Milvus 比 Vearch 快 6.4-27× 的对比一致——Vearch 在 Manu 时代仍是最弱选项。但 Manu 与 Milvus 1.x 论文的对手列表不重合（Manu 不与 SPTAG / 商业系统比，与开源 vector engine 比）。

## Mixed Workload Test (vs Milvus 1.x, [Fig 6])

[guo-2022-manu §5]

固定 collection，逐步插入向量（**2k / 3k / 4k QPS** 三档），同时持续发 search 请求。

| Insertion rate | Milvus 1.x search latency | Manu search latency |
|---|---|---|
| 2k QPS | low | low（与 Milvus 持平） |
| 3k QPS | **明显升高** | low（基本不变） |
| 4k QPS | **violently fluctuates**，~1s peak | low + 稳定 |

**论文论证 [§5]**：
- Milvus 1.x 的 single-writer 在 4k QPS 下 search latency 恶化——index building 与 brute force search 同节点资源争夺
- **Manu 用 dedicated index nodes** → index building 不抢 search 资源 → search latency 几乎不受 insertion 影响

> **wiki 解读**：这正是 [systems/milvus.md] §5 v2.x 章节描述的 "Streaming Node 与 Query Node / Data Node 分离" 设计的实证。**4× lower latency at 4k insertion QPS** 是该架构的关键收益。

## Elasticity Test ([Fig 9], 24h e-commerce 真实 workload)

[guo-2022-manu §5.2 末段]

- 数据集：SIFT100M
- Workload：24-hour 真实 e-commerce search trace（白天峰 / 夜晚谷 / 促销大峰）
- 规则：
  - search latency < 100ms → reduce query nodes 0.5×
  - search latency > 150ms → add query nodes 2×

**结果**：
- Search workload 显著波动（峰比谷高 10×+）
- Manu **自动调整 query node 数** 跟踪负载
- Search latency 全程保持目标范围

→ Manu 的 fine-grained elasticity 设计成功：query nodes 独立 scale，不影响 data / index nodes。

## Scalability Tests ([Fig 10, 11])

### Query nodes scalability ([Fig 10])

固定 SIFT10M / DEEP10M；query nodes 从 2 到 10：

| Index | Throughput growth |
|---|---|
| Manu HNSW | **接近线性** (2-10 nodes 几乎完美 scaling) |
| Manu IVF-FLAT | 接近线性 |
| Vearch HNSW | 平坦（不 scale） |
| Vearch IVF-FLAT | 平坦 |

**论文论证**：Manu 用 segment 在 query nodes 间分布；每 query node 独立处理子集。

### Data volume scalability ([Fig 11])

固定 query node 数；dataset 从 20M 到 100M：

- Throughput 与 dataset 大小**呈倒数关系**（per-vector 工作量基本固定）
- 与 query nodes 数无关

[guo-2022-manu §5.2 末段] **关键观察**：可以通过 **配置 Manu 用更大 segment** 进一步提升 dataset 增长下的 throughput——因为 ANN search complexity 通常 sub-linear w.r.t. segment size。

## 索引构建时间 ([Fig 13])

| Dataset | HNSW | IVF-FLAT |
|---|---|---|
| 20M-100M | linear scaling, 2× | linear scaling, 2× |

→ 两索引时间都随数据量线性。

## Grace time 与 Time tick interval ([Fig 12])

[guo-2022-manu §3.4 + §5.2]

测试 delta consistency 参数 τ（**grace time** = update 延迟容忍）的影响：

| time tick interval | search latency 行为 |
|---|---|
| 200ms | 平滑下降随 grace time 增加（fast-converge） |
| 400ms | 中等 |
| 800ms | **显著抖动**（time-tick 间隔大，等待时间长） |

→ **τ 越大 + time-tick 越频繁** = search latency 越低。但 time-tick 频繁度有 throughput 代价。

## 可信度评估

- **实验设计**：Manu 团队 = Zilliz（[Milvus](../systems/milvus.md) 同公司），但对手 vector engine 都用最新公开版本；硬件统一（AWS m5.4xlarge）；recall 阈值 ≥ 0.8 公允
- **潜在偏向**：
  1. **不与 Faiss / SPANN / DiskANN / 商业系统直接对比**——Manu 论文专注与"开源 vector engine"比，避开了与 algorithm-only 竞品（Faiss, DiskANN）和 SaaS（Pinecone）的正面对比
  2. **数据集仅 10M-100M**——比 Milvus 1.x 论文的 SIFT1B / 12 节点数据小一档；billion-scale 数据未给
  3. **HNSW M=16 / ef=200**——Manu 自家选择，可能 favor 自己实现的 cache 优化
  4. **Vearch / Vald / Vespa 默认参数**——可能 these 系统调优后 closer
  5. **NeurIPS 2021 BigANN winner 数据 (60% recall ↑) 在 §4.4 提及但未在 §5 主 benchmark 出现**
- **复现难度**：中。Manu = Milvus 2.x 全开源（[milvus-io/milvus/tree/2.0](https://github.com/milvus-io/milvus/tree/2.0)）；BIGANN 数据集公开；其他 baseline 全开源
- **场景局限**：
  - L2 + 内积；MIPS-specific 优化（[ScaNN](../concepts/scann.md)）未涉及
  - 维度 96 / 128；现代 768 / 1024-d 未测
  - 单 region；多 region 未测
  - 24h elasticity 实测但 multi-day（如周级数据漂移）未测

## 与其他 wiki 大规模部署对比

| 部署 | 数据 | 索引 | Update | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | freeze | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 多机分片 | static | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | freeze | [douze-2024-faiss-library §7.1] |
| DiskANN @ z840 | 1B | Vamana + PQ + SSD | streamingMerge | [subramanya-2019-diskann] |
| SPANN @ Bing | 1B+ / 千亿+ | HBC + closure + SSD | static | [chen-2021-spann] |
| Milvus 1.x SIFT1B | 1B | IVF_FLAT + HNSW etc | LSM | [wang-2021-milvus] |
| SPFresh @ Azure lsv3 | 1B | SPANN + LIRE | **In-place** | [xu-2023-spfresh] |
| **Manu @ AWS m5.4xlarge** | **100M（实测）/ 1200+ 用户生产** | **HNSW / IVF / SSD multi-LSH** | **delta consistency + stream indexing** | **本论文** |

Manu 是 wiki 已有 8 个工业部署中**首个 cloud-native + delta-consistency + 24h auto-elasticity 实测**的组合。
