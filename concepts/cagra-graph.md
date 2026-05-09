---
title: CAGRA Graph（GPU-native proximity graph）
type: concept
sources: [ootomo-2023-cagra]
related: [hnsw.md, nsg.md, nsw.md, vamana.md, freshvamana.md, proximity-graph.md, warpselect.md, product-quantization.md, ../systems/cagra.md, ../systems/faiss.md, ../systems/milvus.md, ../systems/diskann.md, ../topics/gpu-vs-cpu-ann.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/cagra-vs-hnsw-ggnn-ganns.md]
created: 2026-05-09
updated: 2026-05-09
---

# CAGRA Graph

**TL;DR**: NVIDIA 在 ICDE 2024 提出的 **GPU-native proximity graph**——是 wiki 内**首个 from-first-principles 为 GPU 设计的 graph-based ANN 索引**（之前 GPU graph ANNS 如 SONG / GGNN / GANNS 都是 CPU graph 算法的 GPU 移植）。三大设计特性：(1) **fixed out-degree d**——uniform parallelism + 无 warp divergence，对应 CPU graph 的 variable degree；(2) **non-hierarchical**——random sampling 替代 HNSW 多层结构，让 GPU 高并行度承担"找好起点"任务；(3) **directional**——fixed degree 自然 directed graph。Construction 关键创新是 **rank-based reordering**——用 neighbor list rank 替代 distance-based detourable route counting，省掉 N×d×(d_init-1) 次距离重算，实测**1.9× faster + 同 recall** + DEEP-100M 唯一可行（distance-based OOM）。配套 search 算法 4 个 GPU 优化：warp splitting / forgettable hash table / 1-bit parented node management / single-CTA + multi-CTA dual implementation。**实测 GPU graph build 比 HNSW (CPU 64-core) 2.2-27× 快**；**large-batch search 33-77× 快于 HNSW + 3.8-8.8× 快于 GPU baseline（GGNN/GANNS）**。是 NVIDIA RAPIDS RAFT library 核心实现，[Milvus](../systems/milvus.md) v2.6.x GPU_CAGRA 索引基于此。

## 提出背景

[ootomo-2023-cagra §I, §II-C]

GPU-based ANNS 的研究路径之前都是"adapt CPU graphs to GPU"：

| GPU ANN system | CPU graph base | 主要优化 |
|---|---|---|
| **SONG** [Zhao 2021] | NSW [Malkov 2014] | open-addressing hash table + bounded priority queue + dynamic memory allocation |
| **GGNN** [Groh 2022] | k-NN graph | high-throughput search + fast graph construction |
| **GANNS** [Yu 2022] | NSW + HNSW + k-NN | tailored data structures for GPU + reduced operation time |

**问题**：CPU graphs（HNSW 多层结构 / NSG 单 entry point + variable degree / Vamana α-controlled prune）**算法层面是 sequential traversal-friendly 的**——并不为 GPU SIMT 模型优化。GPU 移植后受限于：
- **Variable degree** 引发 warp divergence + load imbalance
- Multi-layer hierarchy 在 GPU 上不必要（GPU 高并行度可以 random sample 多个起点 + 并行算）
- HNSW/NSG 的 conservative pruning（α=1）减少 graph 平均度数 → GPU 算力闲置

→ **CAGRA 直接为 GPU 设计 graph**——让 algorithm 与 hardware 同步演进。

## 三大设计特性（§III intro）

### Feature 1: Fixed out-degree d

[ootomo-2023-cagra §III "Fixed out-degree"]

**与 CPU graph 的关键差异**：

| 维度 | HNSW / NSG / Vamana | **CAGRA** |
|---|---|---|
| Out-degree | **variable**（per-node 随邻域 density 不同） | **fixed = d** |
| 内存布局 | adjacency list 长度变化 | **每 node 固定 d × edge_id** = uniform |
| GPU traversal | warp divergence（不同 node 不同邻居数） | **uniform parallel computation** |
| Load balance | 不均匀（hub vertices vs leaves） | **统一** |

**Trade-off**：
- 优势：GPU 完美 SIMT mapping；warp 32 thread 同时处理 32 个 node 的邻居算距离不会分叉
- 代价：少量 graph 质量损失（不能为某些 hub vertex 加更多 edges）；CPU single-thread 上略慢

[ootomo-2023-cagra §III] 论文哲学："The graph is search implementation-centric and heuristically optimized. Although the fixed out-degree might decrease theoretical graph quality, the gain on GPU utilization is much higher."

### Feature 2: Non-hierarchical（替代 HNSW 多层）

[ootomo-2023-cagra §III "No hierarchy"]

HNSW 多层结构是 CPU 优化：
- Layer L (sparse): 少节点 + 长程边 → fast zoom-in
- Layer 0 (dense): 全节点 + 局部邻居 → final search
- Greedy descent: 从 top layer 找好 entry → bottom layer 精搜

GPU 高并行度可以**用 random sampling 替代 hierarchy**：
- 随机采 p × d 个起点 + 并行算 distance 到 query
- 选 top-M 作 candidate
- 沿 graph 遍历邻居 + update

→ 不需要 HNSW 多层 + greedy descent；不需要 entry point selection 算法。

### Feature 3: Directional

Fixed out-degree 必然 directed graph（每 node 有 d 个 out-edges 但 in-degree 不固定）。这与 HNSW 的 bidirectional edge 管理不同——CAGRA 不维护 in-edges（让算法更简单）。

## Construction Algorithm（§III-B）

[ootomo-2023-cagra §III-B + Fig 1]

```
Input: dataset D
Output: CAGRA graph G with fixed out-degree d

Stage 1 (parallel on GPU):
  Build initial k-NN graph using NN-Descent
    where k = d_init = 2d or 3d (typical)
  Sort each node's neighbor list by distance ascending
  -> Initial G has high recall but degree d_init > d

Stage 2 (sequential, but algorithmically GPU-friendly):

  a) Reordering + pruning:
     For each node u:
       Count detourable routes for each edge (u, v):
         "Detourable route" = exists w in N_out(u) s.t.
           edge (u, w) → edge (w, v) reaches v with cost < d(u, v)
       Sort edges by detourable route count ascending
       Keep top d edges (those least detourable = most "essential")

  b) Reverse edge addition:
     Build reverse graph G_rev = {(v, u) : (u, v) ∈ G_pruned}
     For each node u in G_rev:
       If |N_out_rev(u)| > d: prune by RNG-style rule
     Take d/2 children from pruned + d/2 from reversed
     Output: final CAGRA graph
```

### 核心创新：Rank-based vs Distance-based reordering

[ootomo-2023-cagra §III-B + Eq 3 + Fig 2]

**Detourable route 定义** [Eq 3]：
```
edge (X → Y) 是 detourable iff
  ∃ Z s.t. (X → Z) ∈ G AND (Z → Y) ∈ G AND
         max(d(X, Z), d(Z, Y)) < d(X, Y)
```

直观：从 X 通过 Z 到 Y 比直达近 → (X → Y) 不重要。

**Distance-based reordering**:
- 计算 detourable count 时需要 distance 信息
- 每 edge 需 (d_init - 1) × distance computations
- N × d_init × (d_init - 1) total distance comps → 极昂贵
- DEEP-100M 上 OOM（distance table 装不下）

**CAGRA innovation: Rank-based reordering**:
- **用 neighbor list rank 替代 distance**——k-NN graph 已按距离排序，rank = position
- "X 到 Y 的 rank" = Y 在 X 的 sorted neighbor list 中的位置
- 等价于 "lower rank ⇒ closer in distance"（approximately）
- **不需要 distance recomputation** → 无 distance table memory

| 维度 | Distance-based | **Rank-based** |
|---|---|---|
| Distance table size | N × d_init entries | **0** |
| Time complexity (per node) | O(d_init³) | O(d_init²) |
| 实测时间（4 dataset 平均） | reference | **1.9× faster** |
| Recall (final CAGRA graph) | reference | **comparable** |
| DEEP-100M | **OOM** | **可行** |

→ Rank-based 是 CAGRA 把 graph optimization 从"distance-bottleneck"解锁到"sort-only"的关键 insight。

### 实测 graph quality

[ootomo-2023-cagra Fig 3 + §III-C Q-A1, Q-A3]

CAGRA optimization (rank-based + reverse edge) 对比 raw k-NN graph：

| Metric | Raw k-NN | + reordering | + reverse edge | **Full CAGRA** |
|---|---|---|---|---|
| 2-hop node count | low | medium | medium | **highest** |
| Strong CC count | high (reference) | medium | low | **lowest (best)** |
| Recall (search) | reference | medium | medium | **highest** |

→ 两个 optimization 都贡献：reordering 主要改 2-hop 可达性；reverse edge 主要改 strong CC。

## Search Algorithm（§IV）

[ootomo-2023-cagra §IV-A + Fig 6]

```
Input: query q, top-k, graph G with fixed out-degree d, parameters M, p
Output: top-k nearest neighbors

Buffers:
  internal top-M list (length M ≥ k)
  candidate list (length p × d)

⓪ Random sampling:
   pick p × d random nodes via PRNG
   compute distances → store in candidate list

Loop until convergence:
  ① Internal top-M update:
     bitonic sort to pick smallest M from internal+candidate
     update internal top-M list
  ② Candidate list index update (graph traversal):
     pick top-p nodes in internal top-M that haven't been parents
     mark them as parents (set MSB bit)
     get their neighbors → fill candidate list (length p × d)
  ③ Distance calculation:
     for each unique (first-time) candidate, compute distance
     skip already-seen candidates (via hash table)

Output: top-k from internal top-M list
```

**Convergence**: 当 ② 步骤所有 internal top-M 都已被 parent → 没新 candidate → 收敛。

## 四个 GPU-specific 优化（§IV-B）

### a) Warp splitting (§IV-B-1)

[ootomo-2023-cagra §IV-B-1 + Fig 8]

NVIDIA warp = 32 threads execute same instruction simultaneously。**问题**：低维数据下 warp 浪费：

```
DEEP-1M: 96-d × 4 byte float = 384 byte = 3072 bits
NVIDIA load instruction: 128-bit per thread → 4096 bits per warp
→ 1024 bits / 4096 = 25% warp 浪费
```

**CAGRA**: 软件层把 warp 分成 "teams"（每 team k 个 thread, k = 4/8/16/32）：

```
team size = 32 (no split): 1 distance comp per warp（适合 GIST 960-d）
team size = 8: 4 distance comps per warp（适合 DEEP 96-d）
team size = 4: 8 distance comps per warp
```

实测 [Fig 8]：
- **DEEP-1M (96-d)**: team size 4 / 8 最优
- **GIST-1M (960-d)**: team size 32 最优（不分）
- → **team size 自适应 dimensionality**

⚠️ Team size 太小（如 2）→ 每 thread 寄存器使用 ↑ → 性能反降。

### b) Forgettable hash table (§IV-B-3)

[ootomo-2023-cagra §IV-B-3 + Fig 9]

**Visited node tracking**: 用 open-addressing hash table（继承 SONG 设计）。**问题**：max_iter × p × d entries 可能装不下 shared memory（A100 ~48 KB/CTA）。

**Solution**：smaller table + periodic table reset：
1. Hash table 容量 typically 2^8 to 2^13
2. 周期性 reset（每 1-4 iterations）
3. Reset 时仅注册当前 internal top-M 内 node

**Effect**：
- Shared memory usage **<4 KB**——所有 NVIDIA GPU 都能 fit
- 多了少量 distance computation（被 forget 的 node 重 evaluate）
- **Recall 不退化**（per [SONG] analysis）

实测 [Fig 9]: forgettable 与 standard hash table 同 throughput 或更高。

### c) 1-bit parented node management (§IV-B-4)

[ootomo-2023-cagra §IV-B-4]

**问题**：跟踪 node 是否已作 parent。Naive 方法 = 额外 hash table → 多 latency。

**CAGRA**: **MSB of node index** 作 parent flag。

```
node_index_with_flag = original_index | (parent_flag << 31)
```

**Cost**：dataset size 上限 = 2^31 - 1 ≈ 2.1B vectors（uint32_t MSB 占用）。**对 1B-scale 仍可用**。

### d) Single-CTA vs Multi-CTA dual implementations (§IV-C)

[ootomo-2023-cagra §IV-C + Fig 7, Fig 10, Table II]

```
                      |  Single-CTA          |  Multi-CTA
Use case              |  large-batch (≥100)  |  small-batch / high recall
Per query             |  one CTA             |  multiple CTAs
Hash table location   |  shared memory       |  device memory
Hash table mgmt       |  forgettable         |  standard
```

**CAGRA dispatch logic** [Fig 7]:
```
if batch_size > b_T (≈ #SMs of GPU):
    use single-CTA  # large batch fully utilizes GPU even with one CTA per query
elif M (internal top-M size) > M_T (≈ 512):
    use single-CTA  # 512+ sort cost dominant on multiple CTAs
else:
    use multi-CTA   # small batch needs multiple CTAs to saturate GPU
```

→ **CAGRA 自动 dispatch**——同一框架 handle batch + single query 两个极端。

## Construction time（§V Q-C1, Fig 11）

[ootomo-2023-cagra §V-A + Fig 11]

DGX A100 (A100 80GB GPU + EPYC 7742 64-core CPU):

| Method | SIFT-1M | GIST-1M | GloVe-200 | NYTimes |
|---|---|---|---|---|
| **CAGRA** | **14.5 s** | **28.9 s** | **25.1 s** | **7.6 s** |
| GGNN (GPU) | 16.0 | 74.8 | 215.9 | 47.0 |
| GANNS (GPU) | 14.6 | 229.7 | 151.9 | 35.8 |
| **HNSW (CPU 64-core)** | 32.3 | 901.1 | 172.2 | 226.3 |
| NSSG (CPU) | 120.5 | 691.6 | 981.7 | 311.7 |

→ **CAGRA build 2.2-27× faster than HNSW**, 1.1-31× than GGNN, 1.0-6.1× than GANNS。

DEEP-100M build: CAGRA **1305.6 s (A100)** vs HNSW **2623.3 s (EPYC 7742)** = **2× faster**。

## 与 wiki 已有 graph 算法对比

### vs [HNSW](./hnsw.md)（CPU graph state-of-art）

| | HNSW | **CAGRA** |
|---|---|---|
| Out-degree | variable | **fixed d** |
| Hierarchy | multi-layer | **non-hierarchical** |
| Initial entry | top-layer descent | **random sampling p × d** |
| Build time SIFT-1M | 32.3 s (CPU 64-core) | **14.5 s (A100)** = 2.2× |
| Build time GIST-1M | 901.1 s | **28.9 s** = **31×** |
| Search recall@10 95% large-batch | reference | **33-77× faster** |
| Search recall@10 95% single | reference | **3.4-53× faster** |
| Hardware | CPU | GPU |

→ CAGRA 是 HNSW 的 GPU-native counterpart——**算法选择因 hardware 而异**。

### vs [NSG](./nsg.md) / NSSG

[ootomo-2023-cagra §V-B Q-C2]

CAGRA paper 用 NSSG (NSG variant) 作 graph quality baseline——**load CAGRA graph into NSSG search implementation 直接对比 graph quality**：CAGRA graph 与 NSSG graph 在 recall-throughput 几乎重合（Fig 12）→ **CAGRA graph quality 与 NSSG comparable**（while build 5-30× faster on GPU）。

NSSG 是 CPU graph state-of-art for ANNS quality；CAGRA 在 GPU 上做到等价 quality + 大幅加速。

### vs [Vamana / DiskANN](./vamana.md)

CAGRA 论文未对比 Vamana——Vamana 是 disk-resident 优化（α-RNG + 长程边）；CAGRA 是 GPU memory-resident 优化（fixed degree + uniform parallelism）。**两者目标 hardware 不同**，没有直接竞争。

理论上 Vamana α=1.2 graph 也可以适配 GPU search（fixed degree → 直接 import to CAGRA search implementation），但 build 仍是 CPU——CAGRA paper §V-B Q-C2 实测 NSG 适配 CAGRA search 也 work。

### vs [FreshVamana](./freshvamana.md)

CAGRA 是 **static graph**——不支持 streaming insert/delete。FreshVamana 是 streaming-ready 但 CPU-only。**两者 orthogonal**。理论上 GPU streaming graph ANN 是 logical work——但 CAGRA 论文不涉及。

## 与 wiki 已 ingest concept 的关系

### 与 [Proximity Graph](./proximity-graph.md)

CAGRA 是 proximity graph 家族的新成员 + 首个 GPU-native：

| Graph 类型 | 提出 | 主要 hardware | wiki coverage |
|---|---|---|---|
| k-NN | classic | CPU | ✓ proximity-graph.md |
| RNG / SNG | classic | CPU | ✓ |
| MRNG | NSG 论文 [Fu 2017] | CPU | ✓ proximity-graph.md "Variants table" |
| HNSW (NSW + skip-list) | [Malkov 2016] | CPU | ✓ |
| Vamana (α-RNG) | [Subramanya 2019] | CPU + SSD | ✓ |
| **CAGRA graph** | **[Ootomo 2023]** | **GPU** | **本 page** |

### 与 [WarpSelect](./warpselect.md)

[Johnson 2017] 同 NVIDIA Faiss 团队的 GPU k-selection。CAGRA search ① 步骤（top-M update）功能 similar——但用 bitonic sort（warp-level / radix-sort）而非 WarpSelect。两者 NVIDIA 不同子团队（Faiss vs RAPIDS）的技术演化：
- WarpSelect: 单 stage k-selection
- CAGRA bitonic sort: integrated into multi-stage graph search

### 与 [PQ](./product-quantization.md)

CAGRA 论文末尾 §V-E 提及 quantization 是 future direction for very large datasets——但 CAGRA 自身**不集成 PQ**。理论上 CAGRA + PQ 可以减少 GPU memory footprint，类似 [Faiss-GPU IVFPQ](../systems/faiss.md) 模式——open work。

## Open Questions

- **CAGRA + Streaming insert/delete (FreshCAGRA)**：CAGRA 是 static graph；FreshVamana 思路（α-augmented RobustPrune）能否扩到 GPU？理论上 fixed-degree graph 的 streaming update 比 variable-degree HNSW/NSG 更难（每 insert 必须 prune 不能 grow degree）——**完全 open**
- **CAGRA + PQ / RaBitQ**：减少 memory footprint 让超 GPU 显存的 dataset 可行；论文 §V-E 提为 future
- **Multi-GPU sharding**：论文 §IV-C 提及但未深入；多 GPU 间 graph 切分 + cross-shard search 协议未涉及
- **CAGRA + filter / multi-vector**：论文未涉及；与 [FilteredVamana](./filtered-vamana.md) / [ACORN](./acorn.md) 整合 logical
- **CAGRA + iterator + RM (VBASE)**：理论上 CAGRA 满足 [Relaxed Monotonicity](./relaxed-monotonicity.md)（greedy graph traversal Phase 1 → 2）；但 GPU batch 与 CPU iterator interface 不易 fit——VBASE 论文 [zhang-2023] 也未集成 GPU
- **CAGRA + α-RNG**：CAGRA 当前 reorder + reverse edge 是 RNG-style 但不显式用 α 参数；α 调节 graph density 的能力 wiki 内 Vamana 已知，CAGRA 论文未深入
- **GPU streaming graph fresh-ANNS**：FreshDiskANN 是 CPU+SSD；GPU equivalent open
- **CAGRA team size 与 dimensionality 关系闭式**：实测 96-d 用 4-8 / 960-d 用 32；中间 dimensionality (200, 256) optimal team size 经验调参
- **CAGRA 与 [Block Shuffling](./block-shuffling.md) GPU equivalent**：GPU memory layout 优化是开放问题——但 GPU device memory random access 比 SSD block read 便宜得多，影响小
- **CAGRA + embedding model 升级**：与 wiki 全 frontier 一致，zero coverage

Cited by: 待 query 引用
