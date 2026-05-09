---
title: Starling vs DiskANN / SPANN on Segment（4 datasets ANNS + RS）
type: benchmark
sources: [wang-2024-starling]
related: [../systems/starling.md, ../systems/diskann.md, ../systems/spann.md, ../systems/milvus.md, ../concepts/block-shuffling.md, ../concepts/vamana.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../topics/disk-vs-memory-ann.md, ../topics/vector-range-query.md]
created: 2026-05-09
updated: 2026-05-09
---

# Starling vs DiskANN / SPANN on Segment

**TL;DR**: Starling SIGMOD 2024 论文 §6 在 4 个 real-world datasets (BIGANN 33M, DEEP 11M, SSNPP 16M, Text2image 5M) 上对比 Starling vs DiskANN vs SPANN——**严格遵循 Milvus segment 配置**（2GB RAM + 10GB disk + 8 vCPU n2-standard-8 NVMe SSD）。**核心发现**：在 segment 约束下 (a) **SPANN 在 Text2image 上严重失败**——8× 复制策略受 10GB 容量限制无法发挥；(b) **DiskANN ANNS recall 同等下 latency 是 Starling 2×**；(c) **Starling RS QPS 比 DiskANN 快 43.9×**（98% 低 latency）；(d) Starling-Vamana / Starling-NSG / Starling-HNSW 三个 variant 全 2× 快于对应 baseline framework——证明 block shuffling + in-memory navigation graph 是 graph-agnostic 的优化。Billion-scale BIGANN 1B 通过 31 segments × 2GB RAM × 10GB disk 跑通，Starling >2× DiskANN at high recall。

## 实验设置

[wang-2024-starling §6.1]

### 数据集

| Dataset | Size | D | Data type | Distance | Query type |
|---|---|---|---|---|---|
| **BIGANN** | 33M | 128 | uint8 | L2 | ANNS + RS |
| **DEEP** | 11M | 96 | float | L2 | ANNS + RS |
| **SSNPP** | 16M | 256 | uint8 | L2 | RS |
| **Text2image** | 5M | 200 | float | IP (inner product) | ANNS |

数据集均为 [big-ann-benchmarks.com](https://big-ann-benchmarks.com/)。每 dataset 限 raw vectors per segment ≤4GB，按 dataset 调 vector count。Query set **not in-database**——验证非 lookup 场景。

### 硬件 + 评估平台

[wang-2024-starling §6.1 + 表]

| 组件 | 规格 |
|---|---|
| **Index building** | n2-standard-64 (Ubuntu 20.04, **64 vCPUs**, 2 TB SSD Persistent Disk) |
| **Search runtime** | n2-standard-8 (8 vCPUs, **375 GB NVMe Local SSD Scratch Disk**) |
| Disk read mode | `o_direct` (**绕过 OS page cache**，公平对比) |
| Default memory budget | 2GB per segment |
| Default disk budget | 10GB per segment |
| Default block size | 4KB |
| Threads | 8（segment-level worker）|
| Repetitions | 3 trials, report average |

### Compared Methods

| Method | 实现 | 备注 |
|---|---|---|
| **DiskANN** | DiskANN reference impl | state-of-the-art baseline |
| **SPANN** | SPANN reference impl | 注意：**8× vector replication 严重受限** |
| **Starling-Vamana (default)** | open-source [zilliztech/starling](https://github.com/zilliztech/starling) | uses Vamana as base graph |

Excluded: HNSW / IVFPQ in-memory（OOM in 2GB budget），LSH / tree-based（curse of dimensionality）。

### Search 参数

[wang-2024-starling §5.1 + §6.1]

| Parameter | Value | 备注 |
|---|---|---|
| Block pruning ratio σ | **0.3** | optimal 跨 4 datasets |
| RS dynamic ratio φ | **0.5** | optimal |
| BNF iterations β | **8** | optimal |
| BNF gain threshold τ | **0.01** | default |
| Search thread count | 8 | 并发 |
| Default ANNS k | 10 | RS r 各 dataset 不同 |

## 主结果

### Result 1: ANNS Performance（Fig 6, Fig 7）

[§6.2 ANNS performance]

QPS vs Recall 曲线（4 datasets, k=10）：

| Dataset | Recall=0.95 latency Starling | Recall=0.95 latency DiskANN | Speedup |
|---|---|---|---|
| BIGANN 33M | **5ms** | 10ms | **2×** |
| DEEP 11M | low | reference | **2×** |
| Text2image 5M | low | reference | **>2× DiskANN, >10× SPANN** |
| SSNPP 16M | omitted | omitted | RS-only dataset |

**关键观察**：
- Starling 在 **Text2image 上比 SPANN >10×** —— SPANN 在 segment 约束下复制不足，性能严重退化
- **Recall 0.95-0.99 区间** Starling consistently 优于 DiskANN
- DiskANN 的 dominant cost 是 disk I/O（92.5% total time）；Starling 用 block shuffling + in-mem nav graph 减少 disk I/O 数

### Result 2: Range Search Performance（Fig 4, Fig 5）

[§6.2 RS performance]

AP vs Latency + QPS vs AP curves（BIGANN 33M, DEEP 11M, SSNPP 16M）：

| Dataset | AP=0.9 QPS DiskANN | AP=0.9 QPS Starling | Speedup |
|---|---|---|---|
| **BIGANN 33M** | 181 | **8690** | **43.9×** |
| **DEEP 11M** | reference | high | **>40×** |
| SSNPP 16M | reference | reference | 接近（query 集中 centroid） |

**关键观察**：
- BIGANN RS **43.9× speedup** 是论文最大数字
- Starling latency reduction up to **98%** under same AP
- DiskANN 用 iterative ANNS 模拟 RS——多次 K' 翻倍重 search → 大量冗余 disk I/Os
- Starling 用 block search + dynamic candidate set 一次性扫到 r 边界

### Result 3: I/O Efficiency（Tab 2）

[§6.3]

Vertex utilization ratio ξ + search path length ℓ on 4 datasets：

| Framework | Metric | BIGANN | DEEP | SSNPP | Text2image |
|---|---|---|---|---|---|
| DiskANN | ξ | 0.0625 | 0.1429 | 0.1111 | 0.2500 |
| **Starling** | ξ | **0.3438** | **0.4429** | **0.4111** | **0.8760** |
| DiskANN | ℓ | 362 | 341 | 100 | 269 |
| **Starling** | ℓ | **182** | 240 | **100** | 167 |

**关键观察**：
- ξ **5-7× 提升**——block shuffling 把 vertex utilization 从 6.25% 拉到 34-87%
- ℓ **0.5-2× 减少**——in-memory navigation graph 提供 query-aware entry → 跨过大部分搜索路径
- Text2image ξ=0.876 接近最优（理论上界 1.0）

### Result 4: Index Cost（Fig 8）

[§6.4]

BIGANN 33M index time + memory cost：

| Component | DiskANN | Starling |
|---|---|---|
| Index processing time | 1214s | **1189s** |
| - Disk graph build | 76.6% | 78.2% |
| - Hot vertices acquisition | 16.7% | n/a |
| - Block shuffling | n/a | 9.5% |
| - In-memory graph | n/a | 5.5% |
| - PQ pre-process | 6.7% | 6.8% |
| Memory cost (search-time) | 1925 MB | **1895 MB** |
| Disk cost (segment) | same | same（graph 本体不变） |

→ Starling 与 DiskANN **几乎同 build cost + 同 memory + 同 disk**——所有性能优势来自**布局重排 + 内存图**而非额外 budget。

### Result 5: Generality across Graph Algorithms（Fig 16）

[§6.7]

Disk-NSG vs Starling-NSG / Disk-HNSW vs Starling-HNSW（BIGANN）：

| Graph algo | QPS @ Recall=0.95 baseline | QPS @ Recall=0.95 Starling | Speedup |
|---|---|---|---|
| NSG | reference | **>2×** | 2× |
| HNSW | reference | **>2×** | 2× |
| Vamana (default) | reference | **2×** | 2× |

→ **Starling 是 framework 而非 algorithm**——block shuffling + nav graph + block search 对 Vamana / NSG / HNSW 全部 work。

### Result 6: Scalability（Tab 3, Fig 15）

[§6.7]

| # Segments | DiskANN QPS RS | Starling QPS RS | Speedup |
|---|---|---|---|
| 1 | 181 | **8690 (48.0×)** | 48× |
| 2 | 42 | **1040 (24.8×)** | 25× |
| 3 | 31 | **687 (22.2×)** | 22× |
| 4 | 23 | **472 (20.5×)** | 21× |
| 5 | 19 | **204 (10.7×)** | 11× |

→ 多 segment 时 Starling 优势收窄但仍 10-48×；ANNS 类似。

### Result 7: Billion-scale（Fig 19, §6.11）

[§6.11]

BIGANN 1B：31 segments × 32GB RAM allocation per node × 10GB disk per segment（2 query nodes total）：

| Framework | Mean I/Os Recall=0.99 | QPS Recall=0.99 |
|---|---|---|
| DiskANN | high | reference |
| **Starling** | 20000+ fewer disk I/Os than DiskANN | **>2× DiskANN at recall>0.96** |

→ 1B BIGANN through segmentation——**与 DiskANN single-server 1B / SPANN Bing 几千亿是第三条路径**。

### Result 8: Block Shuffling Ablation（Fig 9）

[§6.5]

DEEP 11M：

| Layout | OR(G) | Block count to fetch top-1000 NN |
|---|---|---|
| DiskANN (no shuffle) | ~0.06 | 1000 |
| Starling-BNP | mid | ~700 |
| **Starling-BNF** | **0.4429** | **<700** |
| Starling-BNS | highest | lowest |

→ Block shuffling 直接降低 30%+ block 加载数；BNS > BNF > BNP 但 BNF 最经济。

### Result 9: In-Memory Graph Ablation（Fig 10）

[§6.5]

BIGANN 33M：with vs w/o in-memory graph：

| Setting | Disk I/Os Recall=0.95 | QPS |
|---|---|---|
| w/o in-memory graph | reference | reference |
| **w/ in-memory graph** | **20% fewer** | **higher** |

→ In-memory navigation graph 减少 20% disk I/Os（同 recall）通过 query-aware entry。

### Result 10: Block Search Optimizations Ablation（Fig 11）

[§6.5]

BIGANN 33M：三个 computation 优化逐个 ablate：

| Setting | QPS @ Recall=0.95 | Effect |
|---|---|---|
| w/o block pruning | reference | -- |
| w/ block pruning | higher | unnecessary computations 削减 |
| w/o pipeline | reference | -- |
| w/ I/O+computation pipeline | higher | DR/DC parallelism |
| w/o PQ approx distance | many disk I/Os | -- |
| w/ PQ approx distance | **fewer disk I/Os same accuracy** | 不加载邻居 full vector |

→ 三个优化各自贡献；PQ approx distance 影响最大（减少 dominant cost = disk I/Os）。

## 可信度评估

### 实验设计

- ✓ 4 datasets 跨 distance metric (L2/IP), data type (uint8/float), dimension (96-256)
- ✓ Query set 是 not-in-database（避免 lookup 偏向）
- ✓ Hardware spec 详细 + o_direct 绕过 OS cache
- ✓ Open source 代码 (zilliztech/starling)
- ✓ Ablation 充分（每个组件独立 verify）
- ✓ Billion-scale 实证

### 复现难度

- Starling 开源 (zilliztech/starling)
- DiskANN reference impl 公开
- SPANN reference impl 公开
- Hardware n2-standard-8 / n2-standard-64 是 GCP 标准 instance（公开）
- BIGANN / DEEP / SSNPP / Text2image 全公开 dataset
- Hyperparameter (β=8, σ=0.3, φ=0.5) 论文披露

### 偏向

- **作者论文自评 + 同公司 Zilliz / Milvus 团队**——baseline 选择 + 参数 tuning 对 Starling favorable 是潜在风险
- 但 ablation 全 + open source verifiable + multiple datasets + RS 评测扎实
- SPANN 比较是合理（论文 §1 footnote 1 明确 exclusion 理由）；不是 unfair selection

### 数据集偏向

- 4 dataset 全 ≤33M（million-scale）；1B BIGANN 通过 segmentation 实证
- 缺真正 single-segment billion-scale（segment 上限 ~33M 是设计约束）
- Query distribution 假设 not-in-database 接近真实场景

## Open Questions

- **Distributed 多 segment 协调**：单 query node 多 segment OK；多 query node 跨 segment 协调（query coordinator）的 latency 影响未深入
- **Real-time update / concurrent insert**：实测全静态；动态 segment update 时 OR(G) 漂移 + nav graph stale 行为未实证
- **Filter / multi-vector / hybrid query**：Starling 仅 pure ANNS + RS——与 [topics/attribute-filtering.md](../topics/attribute-filtering.md) 联合性能未实证
- **GPU + cache optimization**：§8 future work——未实证
- **更大 segment size (4GB+ disk → 8GB / 16GB)**：实测 Fig 17(a) 显示 disk 容量增加 SPANN 收益 > Starling，但 Starling 仍优势——边界曲线未深入
- **embedding model 升级**：所有 wiki 已 ingest source 一致——zero coverage
- **Quantizer 升级到 [RaBitQ](../concepts/rabitq.md)**：Starling 当前用 PQ short codes for routing；RaBitQ 替换后理论上 routing 决策更准 → 减少 disk reads。两个论文同年发表未联合实证
- **VBASE iterator + Starling disk layer 叠加**：理论可行未实证
