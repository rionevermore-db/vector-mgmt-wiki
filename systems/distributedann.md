---
title: DistributedANN（首个公开 Microsoft Bing production vector search 架构）
type: system
sources: [adams-2025-distributedann, subramanya-2019-diskann, chen-2021-spann, singh-2021-freshdiskann, xu-2023-spfresh]
related: [diskann.md, spann.md, freshdiskann.md, spfresh.md, starling.md, milvus.md, faiss.md, cxl-anns.md, ../concepts/vamana.md, ../concepts/product-quantization.md, ../concepts/freshvamana.md, ../concepts/lire.md, ../concepts/distributedann-cited-frontier.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../benchmarks/spann-vs-diskann-billion.md, ../benchmarks/spfresh-vs-diskann-spann-update.md, ../concepts/disk-distributed-ann-2025-extensions.md]
created: 2026-05-11
updated: 2026-05-21 (lint: link disk-distributed-ann-2025-extensions BatANN/SPI/Gorgeous bundle)
---

# DistributedANN

**TL;DR**: Microsoft Research / Bing 在 ICML 2025 Vector DBs Workshop 发表的**首次公开 Microsoft Bing 当前 production vector search 架构**。是 MSR 谱系 5 代（[DiskANN](./diskann.md) 2019 → [SPANN](./spann.md) 2021 → [FreshDiskANN](./freshdiskann.md) 2021 → [SPFresh](./spfresh.md) 2023 → **DistributedANN 2025**）的 **distributed 演化分支**。**核心声明**：DistributedANN 已**替换** Microsoft Bing 之前的 conventional scale-out 架构（即 SPANN-style clustered partitioning）。**单个 DiskANN graph 跨 1000+ 机器**，通过 **distributed key-value store 作 shared-disk 抽象**——这是 wiki 内首次出现的架构形态："single graph + distributed shared backend"（vs SPANN/Milvus 的"多 partition + 各自 graph"路线）。50B × 384-d int8 vector slice (一个 Bing web index slice)，26 ms p50 latency / 35 ms p99 / >100K QPS。vs Clustered Partitioning（即 SPANN-style）**Table 1 quantified head-to-head**：+7.8pp recall@5 / +4.5pp recall@200 / 6× throughput / 7.5× less IO，但 **latency 输** (26 vs 16 ms p50)——DistributedANN 用 latency 换 throughput + recall。3 个关键 design 改造：(1) compressed vectors 复制到所有 graph nodes 中（10× space amp 换 1 IO per node read）、(2) in-memory head index 作 beam search 起点、(3) **near-data computation**——node scoring service 跑在每个 KV host 上（~6× bandwidth savings）。[adams-2025-distributedann §1-2]

## 与 MSR 谱系其他 4 系统的定位差异

| | DiskANN | SPANN | FreshDiskANN | SPFresh | **DistributedANN** |
|---|---|---|---|---|---|
| Year | 2019 (NeurIPS) | 2021 (NeurIPS) | 2021 (arXiv/preprint) | 2023 (SOSP) | **2025 (ICML Workshop)** |
| 类型 | Graph + SSD | Centroid + posting on SSD | Graph + SSD + streaming | Centroid + posting + LIRE | **Single graph distributed via KV store** |
| Scale | Single-node 1B | Single-node billion (Bing 几千亿 distributed) | Single-node 800M streaming | Single-node 1B in-place | **50B per slice × multiple slices (Bing web)** |
| Distributed? | ✗ (single-node) | ✓ (partition + replica, hash-based dispatch) | ✗ (single-node, future work mentioned) | ✗ (single-node, future work mentioned) | **✓ (single graph, KV-store backed)** |
| Update support | ✗ | ✗ (full rebuild) | **✓ FreshVamana α=1.2 streaming** | **✓ LIRE protocol incremental** | docs 不显式描述（focus on serving，build via stitching） |
| Bing production status | 内部 implied (Microsoft Research) | **2021-2024 主力 production**（论文 §1 明示 deployed in Bing） | implied | implied | **2025 当前 production**（论文 §1 明示 "DISTRIBUTEDANN has replaced conventional scale-out architectures for serving the Bing search engine"） |
| 公开主张 | "1B single node 98% recall" | "hundreds of billions scale" | "graph-based streaming feasible" | "in-place 增量更新" | **"single 50B graph across 1000+ machines, 6× throughput vs partitioning"** |

**MSR 谱系 5 代演化逻辑**：
- **DiskANN (2019)**：static + single-node + graph SSD 路线奠基
- **SPANN (2021)**：static + distributed via partition + centroid SSD 路线 → Bing 2021-2024 主力
- **FreshDiskANN (2021)**：static → streaming 演化分支（graph path）
- **SPFresh (2023)**：static → streaming 演化分支（cluster path）
- **DistributedANN (2025)**：partition + distributed → **single graph + distributed** 演化分支 → **Bing 2025 主力**

> **历史关键转折**：Bing 在 2025 把 production 从 SPANN（partition + distributed）切换到 DistributedANN（single graph + distributed via KV store）。[adams-2025-distributedann §1-2, §5 conclusion]

## 架构图

[per adams-2025-distributedann Figure 1, §2]

```
              ┌─Conventional (e.g., SPANN-style)──┐    │   ┌─DistributedANN──────────────────┐
              │                                    │    │   │                                  │
              │   ┌─Select best N partitions─┐    │    │   │   ┌─Orchestration Service──┐    │
              │   │                          │    │    │   │   │                        │    │
              │   │   ┌──┐  ┌──┐  ┌──┐       │    │    │   │   └────────┬──────────────┘    │
              │   │   │P1│  │P2│  │P3│       │    │    │   │            │ H graph hop batches│
              │   │   └──┘  └──┘  └──┘       │    │    │   │            ▼                    │
              │   │   each: cache + graph    │    │    │   │   ┌─Head Index─┐ ┌─Global Graph─┐│
              │   │         partition (M IO) │    │    │   │   │ Search     │ │ Nodes (BW IO)││
              │   │                          │    │    │   │   │ (in-mem)   │ │ (KV store)   ││
              │   └──────────────────────────┘    │    │   │   └────────────┘ └──────────────┘│
              │                                    │    │   │                                  │
              └────────────────────────────────────┘    │   └──────────────────────────────────┘
                                                        │
              I/O per query: N × M (e.g., 40 × 120)     │   I/O per query: H × BW (e.g., 5 × 128 = 640, vs 4800 for SPANN)
              Recall@5: 83%                             │   Recall@5: 90.8%
              p50: 16 ms (faster!)                      │   p50: 26 ms (trade)
              Throughput: ~15K QPS                      │   Throughput: >100K QPS (6× headroom)
```

3 组件：
- **Orchestration Service**：query 入口；调 head index 取 starting points → 发 H 轮 batch 到 node scoring → 合并 → 返回 top-K
- **Head Index**：top-layers of DiskANN graph 作 separately sharded in-memory ANN（C=2.5B 向量）；提供 beam search 起点
- **Global Graph Nodes (KV store)**：50B graph nodes 分布在 KV store hosts；每 host 跑 node scoring service

## 数据流 / 控制流

### Index Layout 改造（§2.2）

DiskANN single-node 的 index layout（PQ in DRAM + full vector + neighbor IDs on SSD）直接搬到分布式环境会爆 2 个 bottleneck：
- Memory-resident PQ array 单机装不下 50B × 64 byte = 3 TiB
- 每次 graph hop 都要 network roundtrip 整个 node → bandwidth 爆掉

**3 项关键改造**：

**(1) Compressed Vectors Duplicated into Graph Nodes**
- 把 OPQ 压缩 representation **复制到所有该 vector 是邻居的 graph node** 中
- Space amplification ≈ 10× (R=100, d=384, d_OPQ=64, id 8-byte)
- 收益：**每次 graph hop 只需 1 IO**（不是 I × R lookup, 单 query 数万 KV reads）

公式 (Eq 1)：
```
(1+R) × sizeof(id) + d + R × d_OPQ
─────────────────────────────────── ≈ 10× (R=100, d=384, d_OPQ=64, 8-byte id)
       R × sizeof(id) + d
```

**(2) In-Memory Head Index**
- DiskANN single-node 第一批 beam search hop 都命中 node cache（top-layers）→ 减少 SSD roundtrip
- DistributedANN 即使 node cache 命中也仍要 network hop → latency 累加
- 解决：**top-layers of DiskANN graph 上做 BFS 收集 C 个 vector → 单独建 sharded in-mem ANN index** (head index)
- 每个 query 先在 head index 搜 → 结果作 main graph beam search 起点

**(3) Near-Data Computation (Algorithm 1, §2.3)**
- Sequential traversal × log(N) ≈ 6× 计算成本 in 50B graph
- 解决：node scoring service 跑在每个 KV host
- Algorithm 1: 接收 node keys {k_i} + query q_SDC + threshold t → batch read node + compute distance + 仅返回 score（不返回 full node data）
- Eq 2 bandwidth savings：
```
(1+R) × (sizeof(id) + sizeof(score)) + d + d_OPQ
──────────────────────────────────────────────── ≈ 1/6× （pruning further increases savings）
(1+R) × sizeof(id) + d + R × d_OPQ
```

### Orchestration Service (Algorithm 2, §2.4)

```
Input: query q, beam_width BW=128, hops H=5, result_count k=200,
       head_index_result_count k_head=200, candidate_size L=200
1. Encode q with OPQ → q_SDC
2. Search k_head results in head index → insert into candidate heap H_C
3. For i = 1 to H:
   a. Take BW best candidates from H_C → keys K
   b. {R_i}, {C_i} = NodeScoring(K, t, L, q_SDC)  // remote call to KV hosts
   c. Partial merge → top k into H_R, top L into H_C
4. Sort H_R → return R
```

- 状态量小 → 可在 KV host 共置
- Hedged requests (Dean & Barroso 2013) 容忍 partial failure
- Replica health tracking + replica migration

### Build (§3): Graph Stitching

不增量插入 50B vector（too expensive）。改用：
1. **Clustered partitioning** (Wang 2021)：把数据切成 203 partition（一 slice 的 case），每 partition ≈ 200M
2. 每 partition 独立建 DiskANN graph
3. **Union neighbors**：当 vector 出现在多个 partition 的 closure 中 → 取 union of neighbor lists → unified graph
4. **Head index**：BFS top layers of union → 建 conventional sharded in-mem ANN index over result vectors

Quality 略低于完全 incremental 但 build 时间大幅缩短。

## 关键设计决策

### 1. KV store as shared disk abstraction（§2 opening）

> DISTRIBUTEDANN begins with the abstraction of the key-value store as a large shared disk, and then makes modifications to the data and compute placement choices of DISKANN indices in order to make it practical to serve them in this setting.

→ 这是 wiki 内**第一个把 KV store 当作"shared backend disk"的 production vector system**。可类比 Turbopuffer 的 "object storage as primary"——两者都是 **shared backend** 哲学的不同实现：
- Turbopuffer: **object storage** (S3/GCS) primary, NVMe cache
- DistributedANN: **distributed KV store** primary, in-mem caches

但抽象不同：Turbopuffer 用 object storage 自带 LSM API; DistributedANN 用 KV store key-based API + 数据 layout 自主优化（duplicate compressed vectors into graph nodes）.

### 2. Sublinear scaling via single global graph

> Inside a single ANN index, query cost scales with log|X|... across many partitions (assuming a fixed partition size) the cost will scale with the number of partitions. In other words, a system with P partitions has a search complexity of P log(|X|/P), which is much worse than the complexity of one index over the entire dataset X. [§1]

→ **Single graph + distributed KV store ≈ log(|X|) complexity**，partitioned 是 P × log(|X|/P)。50B 数据：
- Single graph: log(50B) ≈ 36 nodes traversed
- 203 partitions × log(247M) ≈ 203 × 28 ≈ 5684 node visits (if naively iterate)
- Even with smart partition selection (top-40 of 203): 40 × 28 ≈ 1120 node visits

DistributedANN 在 graph 算法层有理论优势——前提是 latency × graph hop budget 撑得起。

### 3. Storage amplification trade-off

| | Clustered Partitioning | **DistributedANN** |
|---|---|---|
| SSD space | 270 TiB | **780 TiB (2.9×)** |
| Memory | 18 TiB | **42 TiB (2.3×)** |
| IO per query | 4800 | **640 (7.5× less)** |
| Recall@5 | 83.0% | **90.8% (+7.8pp)** |
| Latency p50 | 16 ms | 26 ms (loses) |
| Throughput | ~15K QPS | **>100K QPS (6×)** |

3× SSD + 2.3× Memory 换 6× throughput + 7.8pp recall + 7.5× less IO——**hardware footprint 没增加**（host 没多）但每 host 用更多 SSD/RAM 容量。

### 4. Reliability via node-level granularity（§4.2）

| | Clustered Partitioning | **DistributedANN** |
|---|---|---|
| Partition failure 影响 | drops 一大块 dataset, dramatic recall drop | **node-level graceful degradation** |
| Replica scaling | per-partition manual + heavy over-provisioning | underlying KV store load balancing |
| 96% availability recall@5 | n/a (dramatic drop) | **87.0% (vs 100% avail 90.8% → 3.8pp graceful)** |

DistributedANN 可以 timeout 单个 node scoring 请求（hedged request）→ recall 稍降但 latency 稳定。Clustered partitioning 失去 partition → 漏掉整片数据 → recall 暴跌。

### 5. Cluster-internal IO distribution (§4 Figure 3)

DistributedANN 在 288 cluster 间 IO 分布**更平**：popular cluster ~180 IO, 但其他 cluster 也都有 IO（graph hop 可跨 cluster）。Clustered Partitioning 在选中的 40 cluster 内每 cluster 固定 120 IO，其他 248 cluster IO=0——load 集中在被选中分区。

→ DistributedANN 全 cluster 利用率高 + 更灵活的 traversal strategy。

## Scale 边界

[per adams-2025-distributedann §1, §4]

- **Production scale**: 50B × 384-d int8 per slice × multiple slices (Bing web index 是 hundreds of billions of vectors total)
- **Host SKU**: 256-768 GiB memory / 5-10 TiB SSD / 200 IOPS/GiB / 32-64 cores / 40 Gbps network
- **Query stats**: p50=26ms / p99=35ms / >100K QPS sustained
- **Search parameters**: H=5 hops, BW=128, R=72 (truncated from full DiskANN R=100 to reduce storage), k=L=200, k_head=200, head_index_size=2.5B vectors
- **Reliability**: 96% node availability → recall@5 ↓ 90.8% → 87.0% (graceful)

### 瓶颈与 future directions（§5.1）

- **Head index CPU bound at 3 replicas** → 需 GPU 加速 head index
- **TCP networking + kernel** → 应改 kernel-bypass (RDMA/DPDK) 降 tail latency；甚至 computational storage 在 KV host 上
- **Space amplification 10×** → 多 vector 共置一个 graph node 可减少
- **Cross-zone latency 2 ms / cross-rack bandwidth oversubscribed** → dense cluster + full mesh network → shared memory pool for compressed vectors
- **HDD tiering for cold tenants (per-user indices)** → DistributedANN 当前为 high-traffic index 设计

## 生产案例

[per adams-2025-distributedann §1, §5]

- **Microsoft Bing web index** — 当前 production architecture: "DISTRIBUTEDANN has replaced conventional scale-out architectures for serving the Bing search engine"
- **Bing 之前的 SPANN-style clustered partitioning production**: 在 2025 切换到 DistributedANN 之前的状态，Table 1 quantified head-to-head 是这个迁移的 evidence
- **Multiple slices of Bing web index**: 每 slice 50B × 384-d int8, hundreds of billions total

### 与 wiki 内其他 production 系统的关系

- **取代 SPANN @ Bing**：SPANN paper [chen-2021-spann §1] 明示 "deployed into Microsoft Bing to support hundreds of billions scale vector search"——这是 **2021 状态**。DistributedANN paper 2025 明示 SPANN-style 已被 DistributedANN 替换。**SPANN "Bing 几千亿规模生产" 仍是有效的历史 statement, 但 SPANN 不再是 Bing 当前 production 主力**。
- **SPANN 仍 active on Vespa OSS**：[per sources/docs/vespa/llms-full.txt §Billion Scale Vector Search] Vespa 实现 SPANN 作为 billion-scale vector search 选项——SPANN OSS path 仍生存，但 Bing internal path 切换到 DistributedANN.
- **vs Turbopuffer (object storage)**：[per sources/docs/turbopuffer §architecture] Turbopuffer 用 SPFresh + object storage primary，**与 DistributedANN 是两种"shared backend"**：object storage (S3 / GCS) vs distributed KV store。前者更便宜 + cold latency 高；后者更快 + 自主 layout 控制。

## Open Questions

- **KV store 具体实现**: 论文不公开 Microsoft 内部 KV store details（Cosmos DB? 内部自研?）；同行复现需要类似 KV store
- **Hundreds of billions total vector**: §1 提到 "hundreds of billions of vectors" 是 multiple slices 累加, 单 slice 50B; 多 slice 间的 query routing / search 协调 docs 不深入
- **Update / streaming support**: 论文 focus on serving, 没明示 incremental update (Microsoft 同组 FreshDiskANN / SPFresh 是 streaming 路径; DistributedANN 是 distributed 路径; **两者是否结合**? streaming distributed graph 是开放 future work)
- **vs FreshDiskANN/SPFresh head-to-head**: 这 3 个 MSR 论文 focus 不同 axis (distributed vs streaming-graph vs streaming-cluster)——DistributedANN 没和 FreshDiskANN/SPFresh 直接对比, 因 axis 不同
- **Graph stitching quality**: §3 提"quality is lower than fully incremental but sufficient"——具体多 lower? recall gap quantified?
- **R=72 vs DiskANN R=100 trade-off**: DistributedANN 截断 neighbors 到 72 (vs DiskANN 100) 以节省 storage——recall 影响未独立量化
- **Bing 切换时机**: SPANN 2021 deployed → 2025 paper 说"has replaced"——具体哪个 quarter 切换的 / 切换过程 / 灰度 / regression rate? 论文不公开
- **GPU 加速 head index**: §5.1 future direction; 但 head index 2.5B vectors 用 GPU memory hierarchy 跑——具体方案 (CAGRA + GPU sharding? 直接 RAFT?) 未明示
- **kernel-bypass network 实测**: §5.1 提 RDMA/DPDK 可改 tail latency——预期 improvement 量级 unknown
- **DistributedANN + RaBitQ**: 当前 OPQ 压缩 (d_OPQ=64) → 若改 [RaBitQ](../concepts/rabitq.md) (unbiased + sharp bound), 是否能减少 space amplification 10× 的 cost? 论文未讨论, 但理论上 RaBitQ 比 OPQ 更紧凑 + bound 更 tight, 可能改变 trade-off
- **vs [Vespa](./vespa.md) SPANN 实测对比**: Bing 选 DistributedANN 取代 SPANN, 但 Vespa OSS 仍在 SPANN——这是因 (a) Vespa workload 不同, (b) Vespa team 还没看到 DistributedANN, (c) DistributedANN 闭源不开源——还是其他原因? 开放
- **GPU-based ANN (CAGRA / BANG / Faiss-GPU) 对比**: §4.3 dismisses GPU——"for our scenario, scalability benefits outweigh GPU advantages"——但 DistributedANN 自己 future direction §5.1 又提 GPU head index. 论文内部 tension

Cited by: 待 query 引用
