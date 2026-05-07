---
title: Milvus（Vector DBMS）
type: system
sources: [wang-2021-milvus, milvus-docs, douze-2024-faiss-library]
related: [faiss.md, diskann.md, spann.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../concepts/woodpecker.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md, ../benchmarks/milvus-vs-prior-sift10m-deep10m.md]
created: 2026-05-07
updated: 2026-05-07
---

# Milvus

**TL;DR**: Zilliz 开源的 **vector DBMS**——不是 ANN library（[Faiss](./faiss.md)）也不是 SSD-resident 算法系统（[DiskANN](./diskann.md) / [SPANN](./spann.md)），而是真正"vector 作为一等公民"的数据库管理系统。**两个时期**：1.x（[SIGMOD 2021 论文](../sources/papers/wang-2021-milvus.pdf)）建在 Faiss 之上 + 单 writer / 多 reader shared-storage；**2.x（v2.6.x，cloud-native 重写）**：四层 disaggregated 架构 + Streaming Node + 自研 [Woodpecker](../concepts/woodpecker.md) zero-disk WAL + DiskANN/SCANN/GPU CAGRA 等纳入索引族。被 300+ 大企业部署（Salesforce / PayPal / Shopee / NVIDIA / IBM / Airbnb / eBay 等）。LF AI 孵化项目。[wang-2021-milvus §1-2; per sources/docs/milvus/site/en/about/overview.md]

> **本 page 区分两个时期**：SIGMOD 2021 论文描述 1.x；后续小节标 "**v2.6.x:**" 的内容来自 [milvus-docs] 反映 cloud-native 重写。两份 source 描述的是同一产品的不同代际。

## 与现有 wiki 系统的定位差异

[wang-2021-milvus Table 1] 的能力六维对比：

| | Billion-scale | Dynamic | GPU | Attr filter | Multi-vec | Distributed |
|---|---|---|---|---|---|---|
| [Faiss](./faiss.md) | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ |
| Microsoft SPTAG | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| ElasticSearch | ✗ | ✓ | ✗ | ✓ | ✗ | ✓ |
| Jingdong Vearch | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ |
| Alibaba AnalyticDB-V | ✓ | ✓ | ✗ | ✓ | ✗ | ✓ |
| Alibaba PASE (PostgreSQL) | ✗ | ✓ | ✗ | ✓ | ✗ | ✗ |
| **Milvus** | **✓** | **✓** | **✓** | **✓** | **✓** | **✓** |

唯一全 ✓。**核心论点**：现有方案要么 library（不管数据生命周期），要么"传统 DB + vector column"（不能为 vector 做深度优化，如 query optimizer、storage engine、CPU/GPU 协同），要么单维度 vector engine（不支持 dynamic / distributed / 多模态查询）。Milvus 把 vector 当 first-class 重新设计完整 stack。[wang-2021-milvus §1, §8]

## 架构图

### 1.x: 三引擎 (SIGMOD 2021)

```
┌──────────────────────────────────────────────┐
│  data science / AI applications              │
├──────────────────────────────────────────────┤
│  SDK / RESTful API（Python / Java / Go / C++）│
├─────────────────────────────────┬────────────┤
│  Query Engine                   │ GPU Engine │
│  ├─ vector similarity search    │ ├─ kernel  │
│  ├─ attribute filtering         │ ├─ co-proc │
│  ├─ multi-vector query          │ ├─ hybrid  │
│  ├─ indexing (quant / graph /   │ └─ multi-  │
│  │   tree based)                │    GPU     │
│  └─ cache- & SIMD-aware         │            │
│     (SSE/AVX/AVX2/AVX512)       │            │
├─────────────────────────────────┴────────────┤
│  Storage Engine                              │
│  ├─ segment (index + data files)             │
│  ├─ bufferpool manager                       │
│  └─ multi-storage (local / S3 / HDFS)        │
└──────────────────────────────────────────────┘
```

[wang-2021-milvus Fig 1]

### v2.6.x: 四层 cloud-native disaggregated

```
┌────────────────────────────────────────────────┐
│ Layer 1: Access Layer (stateless proxy)        │
│ ├─ load balancing (Nginx / K8s Ingress / LVS)  │
│ └─ MPP aggregation + post-processing           │
├────────────────────────────────────────────────┤
│ Layer 2: Coordinator (single active, master-   │
│   slave HA)                                    │
│ ├─ DDL / DCL / TSO management                  │
│ ├─ Streaming service binding                   │
│ ├─ Query topology / load balancing             │
│ └─ Compaction / index-build dispatch           │
├────────────────────────────────────────────────┤
│ Layer 3: Worker Nodes (stateless executors)    │
│ ├─ Streaming Node ─── shard-level "mini-brain" │
│ │   ├─ WAL append + state recovery             │
│ │   ├─ growing data query                      │
│ │   ├─ Query Delegator (forwards to QN)        │
│ │   └─ growing → sealed handoff                │
│ ├─ Query Node ─── load sealed segments         │
│ └─ Data Node  ─── compaction + index build     │
├────────────────────────────────────────────────┤
│ Layer 4: Storage                               │
│ ├─ Meta storage (etcd)                         │
│ ├─ Object storage (MinIO / S3 / Azure Blob)    │
│ └─ WAL storage (Woodpecker / Kafka / Pulsar)   │
└────────────────────────────────────────────────┘
```

[per sources/docs/milvus/site/en/reference/architecture/architecture_overview.md, four_layers.md]

## 数据模型与 segment

### Entity

每个 entity = 一个或多个向量 + 可选非向量属性（数值类）。例：image search 应用中，一个人物 entity 含正脸/侧脸/姿态三个 vector + 年龄/身高属性。[wang-2021-milvus §2.1]

### Segment

**搜索/调度/缓冲的基本单元**，默认 1 GB。一个 segment 内 index 与原始 data 同存（columnar 物理布局：vec₀ 全表、vec₁ 全表）。Milvus 默认仅对 large segment 建索引，user 可手动为任意 segment 建。[wang-2021-milvus §2.3-2.4]

> **wiki 解读**：segment 是 Milvus 与 [Faiss](./faiss.md) / [DiskANN](./diskann.md) / [SPANN](./spann.md) 在 architecture 上的最大不同。前者把 index 视作 single global object（一个 IVF / 一个 Vamana graph）；Milvus 把 index 切成多 segment 并允许 segment 级别的版本/更新/调度。这是 DBMS-style 思维而不是 algorithm-style 思维。

## 数据流 / 控制流

### 动态数据（LSM 模式）

[wang-2021-milvus §2.3] 借鉴 LSM-tree（Apache Lucene 同款）：

1. 写入到内存 **MemTable**
2. 累积到阈值或每秒一次 → flush 为 immutable **segment**
3. 后台 **tiered merge**：相近大小的 segment 合并到更大 segment，直到 1 GB 上限
4. **out-of-place 删除**：删除标记入 segment，merge 时清理

**Trade-off**：写入吞吐高（顺序 append + 异步 flush）vs 读放大（segment 数多时查询要扫多 segment）。

### Snapshot Isolation [§5.2]

每个 query 只看启动时刻的 snapshot——后续写入创建新 snapshot 不影响进行中查询。垃圾回收线程清理无引用 segment。

> **wiki 解读**：这是 wiki 内首次出现的"读写不互相阻塞"语义。[Faiss](./faiss.md) 没有事务模型；[DiskANN](./diskann.md) / [SPANN](./spann.md) 都假设静态数据。Milvus 把 DBMS 的 MVCC 思想搬到向量层。

### 分布式架构 [§5.3]

**Shared-storage**（Snowflake / Aurora 风格）：

```
        ┌──────────────────────┐
        │ Milvus coordinator   │
        │ (HA × 3, Zookeeper)  │
        └──────────┬───────────┘
                   │ metadata + sharding info
   ┌───────────────┼───────────────┐
   ▼               ▼               ▼
┌──────┐  ┌──────────────┐  ┌──────────────┐
│writer│  │ reader₁ (K8s)│  │ reader_n     │
└──┬───┘  └──────┬───────┘  └──────┬───────┘
   │             │                 │
   └─────────────┼─────────────────┘
                 ▼
        ┌──────────────────┐
        │ S3 / HDFS shared │
        └──────────────────┘
```

- **单 writer + 多 reader**——read-heavy 场景；writer 单实例够用（论文明示）
- **Reader 间 consistent hashing 分片**；K8s 自动 restart 失败实例（无 cross-shard 事务，simple）
- **Writer crash 用 WAL 保证 atomicity**；reader stateless
- 计算层只发 log（不发 data）到存储层，类 Aurora

[wang-2021-milvus Fig 5, §5.3]

## 关键设计决策

### 1. 建在 Faiss 之上但全方位改造（§3）

[Faiss](./faiss.md) 提供 IVF/PQ/HNSW 算法核心，Milvus **保留 Faiss 内核但 wrap 出 DBMS 语义**：

- **Cache-aware query partitioning**（§3.2.1）：把 m 个 query 切成 size-s 块让 query+heap 整体落进 L3 cache；线程绑定 data vector 而非 query（Faiss 的 OpenMP 走 query-per-thread）。SIFT1M 实测 1.5×-2.7× 速度提升 [wang-2021-milvus Fig 11]。
- **Runtime SIMD 选择**（§3.2.2）：Faiss 编译时锁定一个 SIMD flag（`-msse4`）；Milvus 把 SIMD 函数 refactor 为 SSE/AVX/AVX2/AVX512 各自源文件，runtime 根据 CPU flag hooking 选择。**这意味着同一 Milvus binary 部署到不同 CPU 都能用最优 SIMD**——多云部署关键。
- **Bigger k on GPU**（§3.3）：Faiss-GPU k 上限 1024（[WarpSelect](../concepts/warpselect.md) 寄存器限制）；Milvus 多轮迭代——第一轮拿 top 1024，记 d_l = 最大距离 + ID 集；第二轮在 [d_l, ...] 区间继续拿 1024，merge——支持 k 到 16384。
- **Runtime GPU 选择**（§3.3）：Faiss 编译时声明 GPU 数；Milvus 启动后任意 server 上跑，segment-based scheduling 把搜索任务分配到可用 GPU——云原生场景下 GPU 弹性扩缩容关键。

### 2. SQ8H：Milvus 自研的 hybrid CPU/GPU 索引（§3.4）

**问题**：Faiss IVF_SQ8 跑 SIFT1B 时数据放 CPU memory（GPU 装不下），按需 PCIe 传到 GPU。两个限制：
- PCIe 利用率低（每 bucket 单独 copy 太碎，1-2 GB/s 实测 vs 15.75 GB/s 上限）
- query batch 小时 GPU 反而比 CPU 慢（数据传输成本大于 GPU 计算收益）

**Milvus 解法（Algorithm 1）**：
- 大 batch（≥1000）+ centroids 装得进 GPU memory：step 1（找 n_probe centroids）GPU + step 2（扫 bucket）CPU。GPU 只存 centroids（数量小），扫 bucket 在 CPU——避免传输 large data
- 小 batch：fallback 到纯 CPU

效果：SIFT1B 上 SQ8H 系统性优于 pure CPU SQ8 与 pure GPU SQ8（Fig 13）。

> **wiki 解读**：SQ8H 是"硬件意识 + IVF 拆步"思路的具体例子。与 [WarpSelect](../concepts/warpselect.md) 的"全在 GPU 寄存器"路线对照——前者是 GPU-friendly kernel，后者是 hybrid scheduling。两者都是工程层 Faiss 改造，但思路对立。

### 3. Attribute filtering 五策略 + partition-based 主张（§4.1）

详见 [topics/attribute-filtering.md](../topics/attribute-filtering.md)。

简言：A=attr-first-vector-full-scan / B=attr-first-vector-search-bitmap / C=vector-first-attr-full-scan / D=cost-based（AnalyticDB-V 的方案）/ **E=partition-based**（Milvus 新提，按高频 filter 属性预分区）。E 比 D 快 13.7×。

### 4. Multi-vector query 双算法（§4.2）

详见 [topics/multi-vector-queries.md](../topics/multi-vector-queries.md)。

**Vector fusion**——仅适用于可分解相似度（内积）：拼接所有 vector + 加权聚合，单次 ANN 解决，3.4-5.8× faster than iterative。
**Iterative merging**——通用，基于 Fagin's NRA + doubling k'。

### 5. 索引家族选择（§2.2 + v2.6.x 扩展）

**1.x（SIGMOD 论文）支持的索引**：
- **Quantization-based**：IVF_FLAT、IVF_SQ8、IVF_PQ（来自 Faiss）
- **Graph-based**：HNSW（Faiss 实现）、**RNSG**（[NSG](../concepts/nsg.md) 变体）
- **Tree-based**：ANNOY（footnote 3 提及）

**显式排除 LSH**——理由："on billion-scale data, LSH approaches have lower accuracy than quantization" [wang-2021-milvus §2.2]。

**v2.6.x 索引家族扩展** [per sources/docs/milvus/site/en/about/limitations.md, about/overview.md]：

| 类别 | 索引 | 备注 |
|---|---|---|
| Brute-force | FLAT | 全表 |
| Quantization-based | IVF_FLAT / IVF_SQ8 / IVF_PQ | 来自 Faiss（同 1.x） |
| Graph-based | HNSW | 仍是默认 |
| **Disk-resident** | **DISKANN** | NEW，集成 [DiskANN](./diskann.md) Vamana + SSD |
| **MIPS / score-aware** | **SCANN** | NEW，集成 Google [ScaNN](../concepts/scann.md) anisotropic loss |
| **GPU** | **GPU_IVF_FLAT / GPU_IVF_PQ / GPU_CAGRA / GPU_BRUTE_FORCE** | NEW，集成 NVIDIA RAPIDS RAFT (CAGRA) |
| **Sparse** | **SPARSE_INVERTED_INDEX** | NEW，BM25 + SPLADE / BGE-M3 学习稀疏 embedding |
| Binary | BIN_FLAT / BIN_IVF_FLAT | Hamming / Jaccard 距离 |

> **2.x 与 1.x 索引家族的根本变化**：1.x 自家最大亮点是 **SQ8H** hybrid CPU/GPU 索引；v2.6.x 文档**未列 SQ8H**——可能被 GPU_CAGRA / GPU_IVF_FLAT 取代。Cloud-native 重写后 GPU 路线全面让位 NVIDIA RAFT 生态。

> **Open**：RNSG 究竟是 [NSG](../concepts/nsg.md)（论文 ref [20] = Fu 2017）还是 Rand-NSG（论文 ref [61] = Subramanya 2019 = [Vamana](../concepts/vamana.md)/[DiskANN](./diskann.md) 早期名）？正文 §2.2 引用 [20]，但生态里 "RNSG" 一名常被解读为 Rand-NSG。论文未澄清。**v2.6.x 文档把 DISKANN 作为独立索引列出**，暗示 RNSG 与 DISKANN 是两个不同实现——这反向支持 RNSG 是 NSG 而非 Rand-NSG。

### 6. v2.6.x: 关键架构改变（cloud-native 重写）

[per sources/docs/milvus/site/en/reference/architecture/*.md]

**a) Streaming Node** —— 新概念，1.x 没有：
- shard-level "mini-brain"，每 shard（vchannel → pchannel）绑定 exactly-one streaming node
- 承担 WAL append + growing data 查询 + growing → sealed handoff
- Query Delegator 在每 streaming node 上运行，负责把单 shard 查询 fan-out 到 query nodes

**b) [Woodpecker](../concepts/woodpecker.md) WAL** —— 新组件：
- 取代 1.x 用的 Pulsar / Kafka 外部 broker
- **Zero-disk** 设计：WAL 直接落 S3 / GCS / MinIO，metadata 走 etcd
- S3 上 750 MB/s 吞吐（Kafka 130 MB/s）
- 两种部署：MemoryBuffer（200-500ms 延迟）/ QuorumBuffer（single-digit ms）

**c) Worker 三角色分离** —— 1.x 是 reader/writer 二角色，2.x 三角色：
- Streaming Node：实时（growing data + WAL）
- Query Node：历史数据（sealed segments from object storage）
- Data Node：离线计算（compaction + index build）

**d) Coordinator 单点 + master-slave HA** —— 1.x 是 3 实例 HA + Zookeeper；2.x 改为单 Coordinator 有 master-slave HA，简化 metadata 一致性

**e) 多租户 4 层隔离**：database / collection / partition / partition-key——1.x 主要在 collection 级隔离

**f) Hot/Cold storage** —— 频繁访问数据放 memory/SSD，冷数据放更便宜存储；tiered storage cache pool

**g) 三种部署模式**：
- **Milvus Lite**：Python lib，edge / Jupyter
- **Standalone**：单 Docker 镜像，所有组件一进程
- **Distributed**：K8s 集群，billion-scale+

**h) 稀疏向量 + 全文 + Hybrid Search**：BM25 native + SPLADE / BGE-M3 学习稀疏 embedding；同 collection 内 dense + sparse 双 vector field + reranker

## Scale 边界

| 配置 | 数据 | 性能 |
|---|---|---|
| 单节点 ecs.re6.26xlarge（104 vCPU, **1.5 TB RAM**） | SIFT1B 全装内存 | IVF_FLAT 高吞吐 [wang-2021-milvus Fig 10a] |
| 12 节点 ecs.g6e.13xlarge（52 vCPU, 192 GB RAM）分布式 | SIFT1B 分片 | 近线性扩展 [Fig 10b] |
| 论文未测 | 100B+ | wiki 未覆盖 |

> **wiki 解读**：论文 single-node baseline 用 1.5 TB RAM 服务器装下 1B SIFT 全数据——这与 [DiskANN](./diskann.md) 主张的"64 GB RAM + SSD"是**对立硬件预算**。Milvus 走 Faiss 路线（全内存 + 量化压缩），不像 DiskANN/SPANN 那样把数据下放 SSD。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

## 与现有 wiki 系统对比

| | [Faiss](./faiss.md) | [DiskANN](./diskann.md) | [SPANN](./spann.md) | **Milvus** |
|---|---|---|---|---|
| 类型 | Library | 算法系统 + open-source impl | 算法系统 + impl | **DBMS** |
| 主要存储介质 | DRAM / GPU HBM | DRAM (PQ) + SSD | DRAM (centroids) + SSD | DRAM 或 + S3/HDFS |
| 动态数据 | ✗ | ✗ | ✗ | **✓ (LSM)** |
| 分布式 | ✗（仅 IndexShards） | ✗ | ✗（论文有但未开源细节） | **✓ (shared-storage)** |
| GPU | ✓（Faiss-GPU） | ✗ | ✗ | **✓ (SQ8H hybrid)** |
| Attribute filter | ✗（仅 IDSelector callback） | ✗（Filtered-DiskANN 是后继工作） | ✗ | **✓ (5 strategies)** |
| Multi-vector | ✗ | ✗ | ✗ | **✓ (fusion + iterative)** |
| 关系 | Milvus 建在 Faiss 之上 | 平行竞品 | 平行竞品 | 整合多 backend 的上层 DBMS |

**关键互补**：[systems/faiss.md §1](./faiss.md) 明确说 Faiss "明确不做 database / service / feature extraction"——Milvus 正是把这些"明确不做"的功能补齐的层。

## 生产案例

[wang-2021-milvus §6, GitHub bootcamp]（1.x 时期）：

- **Qichacha**：100M+ 公司商标搜索（图片相似性）
- **Beike Zhaofang**：房屋户型图相似性检索
- **Apptech**：化学结构搜索（Tanimoto 距离），从小时级降到分钟级
- **十大示范应用**：image / video search、化学结构、COVID-19 dataset、生物多因素认证、QA、cross-modal 行人检索、recipe-food

**v2.6.x 时期** [per sources/docs/milvus/site/en/about/overview.md]：300+ 大企业生产部署，已知名单包括 **Salesforce / PayPal / Shopee / Airbnb / eBay / NVIDIA / IBM / AT&T / LINE / ROBLOX / Inflection**。2022 支持 billion-scale；2023 扩展到 tens of billions。

**Zilliz Cloud**：Milvus 的全托管 SaaS 版本，与开源版同代码 base。

LF AI & Data Foundation 孵化项目（2020-01），Apache 2.0 License。核心贡献者包括 Zilliz、ARM、NVIDIA、AMD、Intel、Meta、IBM、Salesforce、Alibaba、Microsoft 工程师。

## Open Questions

- **RNSG 身份**：[NSG](../concepts/nsg.md) 还是 Rand-NSG ([Vamana](../concepts/vamana.md))？正文 ref vs naming 矛盾，论文未澄清
- **Snapshot isolation 在 distributed shared-storage 下的一致性细节**：论文 §5.2-5.3 描述 single-node 的 snapshot 与 shared-storage 但未深入 read-after-write across reader instance 的具体保证语义
- **Coordinator 单点扩展**：3 实例 HA 但仍是 metadata 单一权威；多 region 部署下 coordinator 跨域复制？
- **Attribute filtering 的 schema migration**：partition-based strategy E 需要预分区，新加 filter 属性时需重组——论文未讨论
- **GPU FPGA hybrid**：§9 提"已经在 FPGA 上实现 IVF_PQ"，但论文没给 FPGA detail
- **Cloud-native 重新架构**：§9 末尾说"正在 architect Milvus as cloud-native"——本论文之后的 Milvus 2.0+ 是**重写**，本论文描述的是 1.x 架构
- **Embedding model 升级处理**：所有动态数据假设向量空间稳定。模型升级（BERT→SBERT）下的 schema migration / re-embedding pipeline 论文未讨论
- **OPQ / RaBitQ / 现代 quantizer**：1.x 论文 quantization 仅 IVF_FLAT / SQ8 / PQ 三种；v2.6.x 加 SCANN（Google ScaNN）但仍未集成 OPQ / RaBitQ 独立索引。wiki 未覆盖
- **Streaming Node 与 SIGMOD 1.x writer 的语义差异**：v2.6.x 文档说 streaming node 是 "shard-level mini-brain"——一个 collection 多 shard 时多 streaming node；1.x 论文是 "single writer + multi reader" 单点 writer。**写入吞吐扩展性根本不同**——但文档未给具体对比数字
- **Woodpecker QuorumBuffer 与 etcd 元数据的故障域耦合**：[per concepts/woodpecker.md] Open Q
- **2.6.x 实测 benchmark**：所有 wiki 现有数字（SIFT10M HNSW 15000 q/s 等）来自 [SIGMOD 2021 论文](../benchmarks/milvus-vs-prior-sift10m-deep10m.md) = 1.x；2.x cloud-native 重写后实测数字 wiki 未覆盖（Zilliz VectorDBBench 是公开 benchmark 但 wiki 未 ingest）
