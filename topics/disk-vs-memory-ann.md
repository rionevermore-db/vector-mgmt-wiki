---
title: Disk vs Memory ANN（SSD 与 DRAM 的 ANN 路线）
type: topic
sources: [subramanya-2019-diskann, chen-2021-spann, jegou-2011-pq, malkov-2016-hnsw, fu-2017-nsg, douze-2024-faiss-library, wang-2024-starling, jang-2023-cxl-anns]
related: [../concepts/vamana.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/woodpecker.md, ../concepts/lire.md, ../concepts/pinecone-serverless-slabs.md, ../concepts/manu-ssd-hierarchical-kmeans.md, ../concepts/vgpq.md, ../concepts/block-shuffling.md, ../concepts/rabitq.md, ../systems/diskann.md, ../systems/spann.md, ../systems/faiss.md, ../systems/milvus.md, ../systems/spfresh.md, ../systems/pinecone.md, ../systems/analyticdb-v.md, ../systems/pase.md, ../systems/starling.md, ../systems/vbase.md, ../systems/cxl-anns.md, ../systems/distributedann.md, ./in-place-vs-out-of-place-updates.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/spann-vs-diskann-billion.md, ../benchmarks/faiss-trillion-scale.md, ../benchmarks/spfresh-vs-diskann-spann-update.md, ../benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md, ../benchmarks/analyticdb-v-vs-twostep.md, ../benchmarks/pase-vs-cube-freddy.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md]
created: 2026-05-07
updated: 2026-05-19 (CXL-ANNS — CXL 解耦内存作为第三条 scale 轴, 关闭 CXL 中间层 Open Q)
---

# Disk vs Memory ANN

**TL;DR**: 十亿+ 向量索引能否单机部署？传统答案是"必须 PQ 压缩到全内存"（[Faiss IVFPQ](../systems/faiss.md) 路径），代价是召回卡在 60-70%。[DiskANN](../systems/diskann.md) 给出新答案：**graph + SSD 全精度 re-rank**，64 GB RAM + SSD 即可达 95%+ 召回。两条路线对硬件、数据规模、recall 上限的取舍不同。[subramanya-2019-diskann §1 + §4.4]

## 问题陈述

ANN 索引在不同存储层级的代价差异巨大：

| 层级 | 容量（典型单机） | 随机访问延迟 | 带宽 |
|---|---|---|---|
| L1 / L2 cache | KB–MB | 1–10 ns | 1+ TB/s |
| DRAM | 数百 GB | 100 ns | 50–500 GB/s |
| **NVMe SSD** | **数 TB** | **几百 μs** | 3–7 GB/s |
| HDD | 10 TB+ | 10 ms | 100 MB/s |
| Network | ∞ | 1+ ms | 10–100 GB/s |

ANN 的搜索过程涉及大量随机访问（图节点跳转 / 倒排表扫描）。**算法是否对随机访问延迟敏感**决定了它能否下放到 SSD。

## 工业方案对比

| 路线 | 代表 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 |
|---|---|---|---|---|
| **全内存 graph** | [HNSW](../concepts/hnsw.md), [NSG](../concepts/nsg.md) | DRAM | 1B 直接 OOM | TB 级 |
| **量化压缩 + 全内存** | [Faiss IVFPQ](../systems/faiss.md) | DRAM | ~62%（IVFOADC+G+P-32 plateau） | 数十 GB |
| **多机分片** | NSG @ Taobao（32 partition × 1/32 数据） | 分布式 DRAM | ~98% | N 台机器 × 数十 GB |
| **GPU brute force** | [Faiss-GPU](../systems/faiss.md) | HBM | 高 | 单卡 ~32 GB |
| **磁盘 + 量化导航 + SSD re-rank** | [DiskANN](../systems/diskann.md) | DRAM (PQ) + SSD (graph + full vec) | **98.68%** | 64 GB |
| **磁盘 + IVF + 全精度 posting list** | [SPANN](../systems/spann.md) | DRAM (centroids + SPTAG) + SSD (full posting list) | **>90% @ ~1 ms** | ~32 GB |
| **磁盘 + IVF + 全精度 + In-place 增量** | **[SPFresh](../systems/spfresh.md)** | DRAM (centroids + version map) + raw NVMe (SPDK) | >0.86 @ ~5 ms（1B stress test） | **持续 ~10 GB**（无 rebuild peak） |
| **SaaS + slab on object storage + adaptive indexing** | **[Pinecone Serverless](../systems/pinecone.md)** | Memtable + cache (memory + local SSD) + slab on object storage | docs 未公开数字 | docs 未公开（auto elastic） |
| **DBMS + SSD-aware hierarchical k-means + LSH replication** | **[Manu (Milvus 2.x)](../systems/milvus.md)** [per concepts/manu-ssd-hierarchical-kmeans.md] | DRAM (centers) + SSD (4KB block, vectors 复制 4-8×) | NeurIPS 2021 winner（baseline +60% recall same QPS） | DBMS 全套 + delta consistency τ |
| **OLAP DB + Pangu 分布式存储 + VGPQ batching** | **[AnalyticDB-V](../systems/analyticdb-v.md)** [per concepts/vgpq.md] | DRAM (HNSW streaming) + Pangu (VGPQ batching, distributed) | 13B records / 30 TB production | SQL 接口 + 4 plan CBO + lambda async merge |
| **OLTP RDBMS (PostgreSQL) + 8KB page-aligned IVFFlat / HNSW** | **[PASE](../systems/pase.md)** | DRAM (PG buffer) + PG storage (8KB pages, contiguous block alloc) | million-scale per instance（Ant Financial production）| PG 全套 OLTP 能力 + iterative compound query |

[subramanya-2019-diskann §1, §4.4]; [douze-2024-faiss-library §5.5 Fig 8]; [chen-2021-spann §4.2]

## 关键洞见 1：算法对随机访问延迟的敏感度

**Graph 算法**的搜索成本 ≈ hops × per-hop time

- per-hop time ∈ DRAM = ~100 ns
- per-hop time ∈ SSD = ~几百 μs（**3-4 个数量级慢**）

朴素把 [HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md) 搬到 SSD 会让 latency 从毫秒级炸到秒级。所以算法必须做两件事之一：

1. **减少 hops**：[Vamana](../concepts/vamana.md) 通过 α>1 + 长程边把 hop 数降 2-3×
2. **批量化 hops**：[DiskANN](../systems/diskann.md) beam search W=4-8 让一次 SSD I/O 拿多个邻居 → round-trips 减半到 1/8

[subramanya-2019-diskann §3.3]

## 关键洞见 2：全精度 re-rank 改变 recall 上限

[Faiss IVFPQ](../systems/faiss.md) 路径的最终距离用 PQ 估计 → recall plateau 在 60-70%（量化失真无法挽回）。

[DiskANN](../systems/diskann.md) 把 PQ 仅用于导航，**最终 ranking 用 SSD 取回的全精度向量**。这把 recall 上限从"PQ 失真极限"提升到 100%。代价：每次邻居读出 4 KB 扇区里**顺手包含**全精度坐标，所以是"免费"的（[subramanya-2019-diskann §3.5]）。

这一模式（DRAM-PQ + SSD-FullPrecision）是 [PQ](../concepts/product-quantization.md) 范式之后的新混合模式。

## 关键洞见 3：两条 SSD 路线 —— Graph vs Inverted File

[DiskANN](../systems/diskann.md) 与 [SPANN](../systems/spann.md) 同期、同公司（Microsoft）、同目标（1B+ 单机 SSD），但走**对立算法路线**：

| | [DiskANN](../systems/diskann.md) | [SPANN](../systems/spann.md) |
|---|---|---|
| 路线 | **Graph + SSD** | **IVF + SSD** |
| 内存放什么 | PQ codes（32 byte/vec） | Centroids（~16% 总向量）+ SPTAG 索引 |
| SSD 放什么 | Vamana graph + 全精度向量（同扇区） | Posting list（全精度，不量化） |
| 是否用 PQ | 是（仅导航） | **否** |
| SSD 访问模式 | 多次小读（每跳一次，beam search 批 4-8） | **少量大读**（K 个 posting list） |
| 90% recall 延迟 | ~3-4 ms | **~1 ms** |
| 公平 benchmark 下 | DiskANN 快慢交替 | **SPANN 在低 latency budget 下系统性领先** [chen-2021-spann §4.2] |

**两条路线的本质差异**：

- **DiskANN**：把 graph 算法的"少跳"思路移到 SSD，每跳成本仍高所以用 PQ 加速决策、用 beam search 批量化 IO
- **SPANN**：把 IVF 算法的"局部扫描"思路移到 SSD，posting list 大小可控所以一次 SSD 大读可拿全部候选 → 不需要量化

**为什么 SPANN 能不用 PQ**：因为 inverted file 的访问模式是 *block-sequential*（一次读一个 posting list），SSD 顺序读带宽足够。Graph 是 *random-pointer-chasing*，每次读小 → IOPS 上限触顶 → 必须用 PQ 减少候选。

详见 [SPANN vs DiskANN benchmark](../benchmarks/spann-vs-diskann-billion.md)。

## 关键洞见 4：DBMS 层的"内存 + 异步刷盘"模式（Milvus LSM segment）

[Milvus](../systems/milvus.md) [wang-2021-milvus §2.3] 给出第三种"内存 + 磁盘"组合：

- **Memory MemTable** 接受新写入
- 阈值或每秒 → flush 为 **immutable segment**（默认 1 GB）持久化到 local FS / S3 / HDFS
- 后台 **tiered merge** 合并相近大小 segment
- Index 可在 segment 级别延迟构建（默认仅大 segment 自动建）

**与 [DiskANN](../systems/diskann.md) / [SPANN](../systems/spann.md) 的根本差异**：DiskANN/SPANN 假设静态数据，索引一次构建后只读；Milvus 的 LSM 模型支持**持续写入 + 周期 flush + 后台 merge**——是 DBMS 视角而非 algorithm 视角。代价是 segment 多时查询要扫多 segment。

> **wiki 解读**：DiskANN/SPANN/Faiss 都是"index = single global object"思路；Milvus 是"index = sharded over segments + LSM merge"思路。前者优化 query，后者同时优化 query + write。

### 进一步：v2.6.x 把 WAL 也搬到 object storage（[Woodpecker](../concepts/woodpecker.md)）

[per sources/docs/milvus/site/en/reference/architecture/woodpecker_architecture.md] Milvus 2.6 自研 [Woodpecker](../concepts/woodpecker.md) 取代 1.x 用的 Pulsar/Kafka 外部 broker，**zero-disk** 设计——WAL 直接写到 S3 / GCS / MinIO，metadata 在 etcd。

**意义对 disk-vs-memory 主题**：
- 不仅 vector index 数据下放 SSD/object storage（DiskANN/SPANN/Milvus segment）
- **WAL 自身也下放 object storage**——彻底无本地磁盘
- S3 backend 实测吞吐 750 MB/s（Kafka 130 MB/s, Pulsar 107 MB/s），延迟 166 ms

这是**第三种"磁盘 vs 内存"层级的反转**：
1. 第一种（DiskANN/SPANN）：vector data 从 DRAM 下放 SSD
2. 第二种（Milvus LSM）：write path 从 in-memory MemTable 周期 flush 到 segment（local FS / S3）
3. **第三种（Woodpecker）：WAL 从 broker local disk 下放 cloud object storage**

每一层都是"用更慢但更便宜/更可靠的存储介质替代更快的"——背后是 cloud-native "stateless compute + 共享存储" 设计哲学的彻底落实。

## 关键洞见 5：内存层级与算法选择的强耦合

| 层级 | 算法偏好 |
|---|---|
| L1/L2 cache | 任何算法都吃缓存友好的数据布局；HNSW prefetch 显式优化 [malkov-2016-hnsw §5] |
| DRAM only | 全图遍历 (HNSW/NSG/Vamana) 或量化驻留 (PQ) |
| DRAM + SSD | **必须 batch I/O + 减少 hop** → DiskANN |
| HBM (GPU) | brute force + fused k-selection ([WarpSelect](../concepts/warpselect.md)) |
| 网络存储 | 几乎无 ANN 方案能正常工作（除非完全 batch） |
| **CXL 解耦内存池** | **全量 graph+向量驻留（不压缩不下放），但需 caching+prefetch 藏 far-memory 延迟 + near-data 距离计算** → [CXL-ANNS](../systems/cxl-anns.md) |

## 关键洞见 6：CXL 解耦内存——第三条 scale 轴（NEW 2026-05-19）

[per systems/cxl-anns.md; jang-2023-cxl-anns §3.1, §6.2]

此前 wiki 内 billion-scale 的两条路线都**牺牲精度或延迟**：

1. **量化压缩 + 全内存**（[Faiss IVFPQ](../systems/faiss.md)）：recall 卡 60-70%
2. **hierarchical SSD/PMEM**（[DiskANN](../systems/diskann.md) / SPANN / HM-ANN）：storage 访问占 query latency 87.6%（CXL-ANNS 实测），DiskANN/HM-ANN 比无限-DRAM oracle 差 29.4×/64.6× latency

[CXL-ANNS](../systems/cxl-anns.md)（KAIST + Panmnesia, USENIX ATC 2023）给出**第三条 scale 轴——不换更慢介质，而是换"更多内存设备"**：把全量 billion-point 数据集（graph + embedding table，不压缩）放进 **CXL 解耦内存池**（Type-3 EP，协议上限 4095 EP / 4 PB）。

**核心矛盾与解法**：

| | naive CXL 内存池（`Base`） | CXL-ANNS（4 机制后） |
|---|---|---|
| vs oracle latency | **慢 3.9×**（每访问要 RC memory⇄flit 转换；graph traverse +2.6×、distance calc +4.3×） | **低 68% latency / 高 3.8× throughput** |
| vs SOTA billion-scale | — | **111.1× QPS / 93.3% lower latency**（vs PQ/DiskANN/HM-ANN） |

4 机制：(1) relationship-aware caching（按 entry-node hop 距离缓存 2-3 跳内热点）；(2) ANNS-aware prefetch（82.3% 访问来自 candidate array → 提前一轮取）；(3) **EP-side 近数据距离计算 + vector sharding**（DSA 在内存侧算距离，数据传输削 73.4×、距离计算降 119.4×）；(4) urgent/deferrable 依赖松弛（CXL CPU 不再 42% 空等）。

**对 disk-vs-memory landscape 的意义**——现在有**三条 scale 轴**：

| 轴 | 代表 | 换什么 | 代价 |
|---|---|---|---|
| 慢介质（SSD/PMEM） | DiskANN / SPANN | 内存 → SSD（量化导航 + SSD re-rank） | latency ↑（storage 87.6%） |
| 分布式 | [DistributedANN](../systems/distributedann.md) | 单机 → 1000+ 机器（single graph + KV store） | latency ↑（26 vs 16ms p50） |
| **解耦内存（CXL）** | **[CXL-ANNS](../systems/cxl-anns.md)** | **单机 DRAM → CXL 内存池（全精度不压缩）** | **需 CXL 硬件 + 软硬协同藏 far-memory** |

→ CXL 路线的独特点：**唯一同时做到 billion-scale + 全精度无损 + 比 oracle 还低延迟**——但前提是有 CXL 2.0+ 解耦内存硬件（论文用 16nm FPGA 原型 + gem5 验证，无商用实测）。瓶颈也从"存储容量"转移到"EP-side PE 算力"——scale-out 是加 EP/host 而非加 SSD。详见 [systems/cxl-anns.md](../systems/cxl-anns.md)。

## 工业方案适用边界

| N | DRAM 预算 | 推荐 |
|---|---|---|
| < 10M | 任何 | 全内存 graph（HNSW / NSG） |
| 10M–1B | 内存富余（>500 GB） | Faiss `IVF_HNSW,Flat` |
| 10M–1B | 内存吃紧（<200 GB） | Faiss `IVFPQ`（接受 ~70% recall）/ Faiss + GPU |
| **1B+** | **64 GB + SSD** | **DiskANN** |
| 1B+ | DRAM 极便宜 / 多机 | 多机 graph 分片（Taobao 模式） |
| 100B+（trillion） | 任意 | 分布式 mmap + 极端压缩（[Faiss trillion-scale](../benchmarks/faiss-trillion-scale.md)） |
| **vector DBMS segment（≤33M / 2GB RAM / 10GB disk）** | **per-segment 严格约束** | **[Starling](../systems/starling.md)（block shuffling + nav graph）** |

## 关键洞见 4：vector DBMS segment 模型 ≠ single-server 假设（NEW）

[wang-2024-starling §1, §6.9]

之前 wiki 内 disk-resident 系统（[DiskANN](../systems/diskann.md) / [SPANN](../systems/spann.md)）的 hidden assumption 是 "**single-server 大磁盘 + 大内存 budget**"——DiskANN 64 GB RAM + 几 TB SSD；SPANN Bing 几千亿用 single-cluster 大磁盘 + 跨机 replicate。

**vector DBMS 工程现实是不同**：
- [Milvus](../systems/milvus.md) / [Manu](../systems/milvus.md) / 大多数现代 vector DB 用 **segment** 抽象——每 segment ~2GB RAM + ~10GB disk hard limit
- 一台 query node **多 segment** 共享 RAM/disk
- Distributed + load balancing + fault tolerance 都在 segment 粒度而非 server 粒度

→ Single-server 假设的优化在 segment-level **失败**：
- **SPANN**：closure clustering 复制 up to 8× → 33M × 8 = 264M storage 远超 10GB cap → **不可行**
- **DiskANN**：data locality OR(G) ≈ 0（94% block 浪费）+ 长 search path（362 hops）→ **高 latency**

[Starling](../systems/starling.md) 是 wiki 内**首个 explicit 接受 segment-level 约束作为 design first principle** 的 disk graph framework：
- Block shuffling（NP-hard 优化数据 locality 在 4KB block 内）
- In-memory navigation graph（采样 <10% vector 减少 search path）
- Block search + 三个 computation 优化

实测 BIGANN 33M segment：Starling ANNS 2× 快于 DiskANN，RS **43.9× 快**。

→ disk-vs-memory landscape 现在有**第三层**："single-server"路径（DiskANN/SPANN）vs"segment-level"路径（Starling）；两者都属"DRAM + SSD 混合"但 budget 假设不同。详见 [systems/starling.md](../systems/starling.md) 与 [benchmarks/starling-vs-diskann-spann-on-segment.md](../benchmarks/starling-vs-diskann-spann-on-segment.md)。

## Open Questions

- **GPU + SSD 混合**：当前 [Faiss-GPU](../systems/faiss.md) 是全 HBM，[DiskANN](../systems/diskann.md) 是 CPU + SSD。两者结合（PQ in HBM + graph on NVMe）未在文献覆盖。
- **网络存储 ANN**：所有"disk-resident"分析假设本地 NVMe；远程块设备 / 对象存储下 latency 完全不同。
- ~~**持久内存（CXL、Optane）作为中间层**：DRAM 与 SSD 之间出现新的存储层；ANN 算法适配未在 wiki 任何 source 覆盖。~~ **2026-05-19 ingest [jang-2023-cxl-anns] 部分解** — [CXL-ANNS](../systems/cxl-anns.md) 覆盖 CXL 解耦内存作为 ANN scale 轴（caching+prefetch+near-data 算距离藏 far-memory）。**仍 open**：Optane PMEM 作为中间层的 ANN 适配（CXL-ANNS 把 PMEM 当 hierarchical baseline 而非自身介质）；CXL.mem 与 PMEM 作为内存层级的边界未系统化。
- **SSD 寿命与 wear leveling**：高 QPS ANN 服务对 SSD 是持续随机读负载；写入压力低但寿命经济性需要量化。
- **CAGRA / GGNN 等 GPU graph 索引**：把图算法移到 GPU，是与 SSD 路线平行的另一种 scale-out 思路。wiki 尚未 ingest。
- **DiskANN vs SPANN 的最终归宿**：两条路线各自有边界（DiskANN 在高 latency budget 下追平、SPANN 在 query 难度极不均时退化）；最终是融合方案（HBC + graph）还是路线分化是开放问题。

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md), [queries/giga-scale-sharding.md](../queries/giga-scale-sharding.md)
