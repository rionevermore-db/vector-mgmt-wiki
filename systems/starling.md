---
title: Starling（Zilliz/Milvus team segment-level disk-resident graph framework）
type: system
sources: [wang-2024-starling]
related: [milvus.md, diskann.md, spann.md, faiss.md, vbase.md, pase.md, ../concepts/block-shuffling.md, ../concepts/vamana.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../concepts/rabitq.md, ../concepts/relaxed-monotonicity.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/topk-vs-iterator-model.md, ../topics/vector-range-query.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md]
created: 2026-05-09
updated: 2026-05-09
---

# Starling

**TL;DR**: Zilliz / Milvus team 在 SIGMOD 2024 提出的 **I/O-efficient disk-resident graph index framework**——**专为 vector DBMS 的 data segment 抽象设计**（每 segment ~2GB RAM + ~10GB disk capacity，holding 33M vectors at 128-d）。在 segment 约束下 [DiskANN](./diskann.md) 高 latency（disk I/O 占 92.5%）、[SPANN](./spann.md) 严重超 disk 容量（每向量 8× 复制）。Starling 三个核心贡献：(1) **[Block shuffling](../concepts/block-shuffling.md) 数据布局**——NP-hard 问题 + 3 个启发式算法把 OR(G) 从 DiskANN 的 ≈0 提升到 0.34-0.87；(2) **In-memory navigation graph**——采样 <10% vertex 建 in-memory graph 找 query-aware entry points，搜索路径 ℓ 减半；(3) **Block search strategy**——block 内一次性处理所有 vertex + block pruning + I/O+computation pipeline + PQ approximate distance。**实测 RS QPS 比 DiskANN 快 43.9× / 98% 低 latency；ANNS 2× faster；保 same recall**。原生支持 ANNS + Range Search + 任意 graph base ([Vamana](../concepts/vamana.md) / [HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md))。**§8 conclusion 明示 Future work: integrate Starling into Milvus**——是 [Milvus](./milvus.md) 下一代 segment-level disk graph index 的"正统候选"。GitHub: `zilliztech/starling`. [wang-2024-starling §1, §6, §8]

## 与 wiki 现有系统的定位差异

[per topics/disk-vs-memory-ann.md, wang-2024-starling §1 + Fig 1]

**关键 Insight**：vector DBMS 工程现实**不是单机大磁盘**——而是**多 segment per machine**。每 segment ~2GB RAM + ~10GB disk capacity 严格限制下，single-server 假设的 disk 系统全失败：

| | [DiskANN](./diskann.md) | [SPANN](./spann.md) | **Starling** |
|---|---|---|---|
| 设计 target | 单机大磁盘 1B SIFT | 单机大磁盘 + 几千亿 Bing | **Milvus segment ~2GB RAM + ~10GB disk** |
| Single-machine 1B 实证 | ✓ 64 GB RAM | ✓ Bing | ✓ 31 segments × 10GB |
| Segment-level 实证 | ✗（latency 高）| **✗（每向量 8× 复制超容量）** | ✓ |
| Index storage 模型 | full disk graph + DRAM PQ + DRAM hot vertices | DRAM centroids + SSD posting list（**复制 up to 8×**） | **DRAM nav graph + DRAM PQ + reordered SSD graph** |
| 数据 locality 优化 | none（ID-consecutive layout, OR(G)≈0） | closure clustering（cluster 内 locality） | **block shuffling**（OR(G) 0.34-0.87） |
| Query 优化 | beam-search + DRAM PQ rerank | query-aware dynamic pruning | **block search + pruning + pipeline + PQ** |
| Range Search | NA（needs iterative ANNS）| NA | **原生支持** |
| Graph 算法 portability | Vamana fixed | NA（不是 graph）| **Vamana / NSG / HNSW 全可** |
| 集成路径 | Microsoft / open source / DiskANN as is | Microsoft Bing 闭源（SPTAG 库开源）| **Milvus 集成 (§8 future work)** |

**核心论点**：当数据库工程现实是 segment 时，single-server disk 系统假设全部失效；Starling 是 **first work** 把 segment-level constraint 作为 disk graph design first principle。

## 架构图

[wang-2024-starling Fig 2(c)+(d)]

```
┌──────────────────────────────────────────────────┐
│  Single Segment（~2GB RAM + ~10GB disk）         │
├──────────────────────────────────────────────────┤
│  In-Memory                                       │
│  ┌──────────────────────────────────────────┐   │
│  │ Navigation Graph (sampled <10% vertices) │   │
│  │   - HNSW / NSG / Vamana                  │   │
│  │   - Find query-aware entry points        │   │
│  │   - Reduces search path length ℓ by 2×   │   │
│  ├──────────────────────────────────────────┤   │
│  │ PQ short codes（all vectors）            │   │
│  │   - For routing decisions                │   │
│  │   - Avoid loading full neighbors         │   │
│  ├──────────────────────────────────────────┤   │
│  │ vID → blockID mapping                    │   │
│  │   (post block shuffling)                 │   │
│  └──────────────────────────────────────────┘   │
├──────────────────────────────────────────────────┤
│  On-Disk (NVMe SSD via o_direct)                 │
│  ┌──────────────────────────────────────────┐   │
│  │ Reordered disk-based graph               │   │
│  │   - Block shuffling (BNF default)        │   │
│  │   - OR(G) 0.34-0.87                      │   │
│  │   - 4KB block, 16 vertex/block typical   │   │
│  │   - vertex = vector + neighbor IDs       │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

## 数据流 / 控制流

### Index Building

[wang-2024-starling §4 + Fig 8(a)]

```
Input: vectors V in segment

1. Build disk-based graph G(V,E)（用 Vamana / NSG / HNSW 算法之一）
   - Time: T_disk_graph (bottleneck)
2. Block shuffling on G
   - 用 BNF (default) iterative β=8 轮
   - Time: T_shuffling ~9.5% of total
3. Sample ~10% vertices V'
4. Build in-memory navigation graph on V'
   - Time: T_memory_graph ~5.5%
5. Pre-process PQ short codes for all vectors
   - Time: T_PQ
6. Persist: disk graph (reordered), vID→blockID map, PQ codes,
            in-mem navigation graph

Total: T_Starling = T_disk_graph + T_shuffling + T_memory_graph + T_PQ
```

实测 BIGANN 33M：T_Starling = 1189s vs DiskANN 1214s（基本持平）。

### ANNS Query

[wang-2024-starling §5.2 Algorithm 2]

```
Input: query q, top-k

1. Vertex search on in-memory navigation graph
   → Find entry points S（close to q without disk I/O）
2. Initialize candidate set C = S, result set R = S
3. Sort C by PQ approximate distance to q
4. Loop while C has unvisited vertex u:
   a. Read block B from disk containing u
   b. Compute exact distance for u (full-precision vector)
   c. Update R with u; add u's neighbor IDs to C
   d. Block pruning: pick top-((ε-1)·σ) vertices in B \ {u} by PQ dist
      → Add their neighbor IDs to candidate B'
   e. Pipeline: next DR for v = top-1 unvisited in C ||  DC of vertices in B'
   f. PQ-based routing: use PQ codes to rank C
5. Return top-k from R sorted by exact distance
```

**核心区别 vs DiskANN**：
- DiskANN: 每 hop 一个 vertex → 一次 disk I/O；94% loaded data 浪费
- **Starling**: 每 hop 加载一个 block (4KB, ~16 vertex) → 一次 I/O 处理多 vertex；vertex utilization ξ 0.34-0.87

### Range Search Query

[wang-2024-starling §5.3]

```
Input: query q, radius r

1. Same vertex search to find entry points
2. Maintain candidate C, result R, kicked-out P
3. Block search loop:
   a. If |R|/|C| > φ=0.5: double |C|, restart with closer-vertices in P
   b. Standard block search update C, R
4. When C exhausted, return R
```

**关键**：动态扩展 candidate set——不同 query 的 RS 结果数量差异大（同 r 可能 0 个也可能 1000 个）；固定 K 反复迭代浪费。

→ Starling 是 wiki 内继 [VBASE](./vbase.md) 之后**第二个原生支持 vector range search 的 disk-resident 系统**——VBASE 在 query engine 层 (RM iterator)，Starling 在 disk index 层 (block search + dynamic candidate)。详见 [topics/vector-range-query.md](../topics/vector-range-query.md)。

## 关键设计决策

### 1. 接受 segment-level constraint 作为 design first principle（§2.2, §6.9）

[wang-2024-starling §2.2 + §6.9]

- 每 segment ~2GB RAM + ~10GB disk hard limit（Milvus default）
- 一台 query node 多 segment 共享内存
- **不假设 single-server 大磁盘**——这与 DiskANN/SPANN 的 hidden assumption 根本不同

**Trade-off**：
- 优势：与现实 vector DBMS 工程严格契合；segment 模型支持分布式 / 容错 / load-balancing
- 代价：算法选择被严格 budget 限制；某些 single-server 优化（如 SPANN closure replication）不可行

### 2. Block Shuffling NP-hard 问题接受 heuristic（§4.1）

[wang-2024-starling Theorem 4.1 + §4.1]

详见 [concepts/block-shuffling.md](../concepts/block-shuffling.md)。

- BNP O(|V|): 一遍扫描，最快
- **BNF O(β·o·|V|)（default）**: iterative，平衡
- BNS O(β·o³·ε·|V|): NN-Descent style，最优 OR(G)

OR(G) 提升 5-7× → ξ 提升 5-7× → disk reads 显著减少。

### 3. In-Memory Navigation Graph 减少 search path（§4.2）

[wang-2024-starling §4.2]

采样 <10% vector 建 in-memory graph：
- HNSW（多层）/ NSG / Vamana 都可
- 找 query-aware entry points（distance to q 小）
- Disk graph search 从 entry points 启动，**ℓ from 362 hops → 182**

**Trade-off**：
- 优势：减半 search path → 减半 disk I/O
- 代价：~5.5% extra index processing time + 占 in-memory budget（与 PQ codes 共享 2GB）

> 论文 §7 比较 mmap：试过 mmap 但 page cache miss → 更多 disk I/O；最终用显式 in-memory graph + o_direct read。

### 4. Block Search + 三个 computation 优化（§5.1）

[wang-2024-starling §5.1]

| 优化 | 描述 | 效果 |
|---|---|---|
| **Block pruning** (σ=0.3 默认) | block 内按 PQ dist 排序，仅 check top-(ε-1)·σ vertex 的邻居 | 减少 distance computation |
| **I/O + computation pipeline** | DR (disk read) || DC (dist compute) parallel for 不同 vertex | 提升 QPS（Fig 11(b)） |
| **PQ-based approximate distance** | 邻居距离用 DRAM PQ codes 估计，避免加载 full-precision | 大幅减少 disk reads（Fig 11(c)） |

实测 BIGANN：disk I/O 占总时间从 DiskANN 的 92.5% 降至 Starling 的 57.7%——**计算与 I/O 平衡**。

### 5. 任意 graph algorithm 兼容（§4 + §6.7）

Starling 是 **framework**——graph 构造 step 1 可用任何算法：

| Variant | 底层 graph |
|---|---|
| **Starling-Vamana (default)** | [Vamana](../concepts/vamana.md)，与 DiskANN 同 graph 算法 |
| **Starling-NSG** | [NSG](../concepts/nsg.md)，static index 友好 |
| **Starling-HNSW** | [HNSW](../concepts/hnsw.md) layer-0 + upper layers as in-memory navigation graph |

实测 6.7：三个 variant 全部 2× over baseline framework（Disk-Vamana / Disk-NSG / Disk-HNSW）——**block shuffling + in-memory nav graph + block search 是 graph-agnostic 的优化**。

## Scale 边界

[wang-2024-starling §6.7, §6.11]

| 配置 | 数据 | 实测 |
|---|---|---|
| 单 segment 2GB RAM + 10GB disk | BIGANN 33M × 128 uint8 | recall=0.95 latency 5ms; RS AP=0.9 QPS 8690 |
| 5 segments × 2GB | BIGANN（同 dataset） | 持续 2-48× over DiskANN |
| 31 segments × 2GB / 10GB | **BIGANN 1B** | **>2× over DiskANN at recall>0.96** |
| Distributed multi-machine | 论文未深入 | future work（§8） |

> **wiki 解读**：1B BIGANN 是最大实证——通过 segment 切分实现而不是 single-segment scaling。这与 [DiskANN](./diskann.md) 的 "single-server 1B" / [SPANN](./spann.md) 的 "Bing 几千亿 + bin-packing" 是**第三条 billion-scale 路径**：segment-level partition + 同一台机器多 segment 共享。

## 与 wiki 已有系统的对比

### 与 [DiskANN](./diskann.md)（同 graph + SSD 路径）

[wang-2024-starling §6.2-6.3]

| | DiskANN | **Starling** |
|---|---|---|
| 数据 locality 优化 | none（ID-consecutive 默认 layout） | **block shuffling**（OR(G) 5-7× 提升） |
| Search path 优化 | none（random/fixed entry） | **in-memory nav graph**（ℓ 减半） |
| Search granularity | per-vertex | **per-block**（4KB unit） |
| Range Search | iterative ANNS approximation | **原生支持** |
| Vertex utilization ratio ξ | **0.0625-0.25**（94% block 浪费） | **0.34-0.87** |
| BIGANN 33M ANNS recall=0.95 latency | 10ms | **5ms** |
| BIGANN 33M RS AP=0.9 QPS | 181 | **8690 (43.9×)** |
| 1B BIGANN | 1100 GB peak memory @ build (5+ days) | **31 segments × 32GB** (per node) |

→ Starling 在 **same disk graph** 之上做 layout 重排——Step 1（构建 graph）与 DiskANN 同；Step 2（block shuffling）和 Step 3（nav graph）是 Starling 增量。**Starling = DiskANN + block shuffling + in-mem nav graph + block search**。

### 与 [SPANN](./spann.md)（segment-level 不可行）

[wang-2024-starling §1, §2.2]

```
SPANN 假设：vector × 8 复制（boundary closure）→ 33M × 8 = 264M storage
            + posting list 12-48KB × N → 远超 10GB segment cap
```

→ SPANN 的"closure clustering 跨 cluster 复制"在 segment-level 直接 break。Starling 论文 §1 footnote 1 明确"We exclude SPANN as it duplicates each vector up to eight times, which far exceeds the capacity of the data segment"。

实测 Text2image 对比 SPANN：Starling **>10× SPANN** at high recall——SPANN 在 segment 约束下无法 replicate enough。

### 与 [Milvus](./milvus.md)（Zilliz 同公司，集成路径）

[wang-2024-starling §8 + 作者 Charles Xie + Rentong Guo Zilliz]

- Milvus 现有 segment 模型（[wang-2021-milvus §2.3] + [guo-2022-manu §3]）天然 fit Starling
- v2.6.x docs 已列 DISKANN 索引——Starling 是 next-gen disk graph framework，但**当前 Milvus release 未集成**
- §8 conclusion: "We will also integrate Starling into Milvus for distributed optimization"——**logical roadmap**

→ Starling **是 Zilliz 的产品技术栈** future——与 [Manu](./milvus.md)（Milvus 2.x 学术形态）、[Woodpecker](../concepts/woodpecker.md)（zero-disk WAL）一道是 Milvus 体系的"零部件"。

### 与 [VBASE](./vbase.md)（不同层正交 disk 优化）

[wang-2024-starling §7]

| | VBASE | **Starling** |
|---|---|---|
| 优化层 | **query engine layer**（PostgreSQL Volcano 集成） | **disk index layer**（segment-level data layout） |
| 攻击点 | TopK speculation / K' prediction | data locality / search path |
| Range Search | iterator + RM 自然支持 | block search + dynamic candidate |
| 集成关系 | 集成 SPANN 在 query engine | 替代 DiskANN 在 disk graph |
| 是否兼容 | **理论上可叠加**——VBASE engine + Starling disk layer | n/a |

→ 两个 SIGMOD/OSDI 同期工作攻击 disk-resident graph 的不同层；理论上可叠加但 wiki 内 zero coverage 实证。

### 与 [Faiss](./faiss.md)（库 vs framework）

Starling 是**完整 framework**（包含 build pipeline + search engine + parameters）；Faiss 是 library（提供 building blocks）。两者关系类似 [Milvus](./milvus.md) 与 [Faiss](./faiss.md)。

理论上 Faiss 可以集成 block shuffling 作为 IndexAmRoutine-like utility——但需要把 graph index 拉到 disk-resident（Faiss 主要 in-memory）。

## 生产案例

Starling 是 **Zilliz 学术 prototype + 产品 roadmap**——GitHub `zilliztech/starling`，Apache-2.0。

- **当前状态**：开源 reference implementation；未集成进 Milvus release
- **Roadmap**：§8 明示 "integrate Starling into Milvus for distributed optimization"——future Milvus version 候选 disk graph index
- **数据点**：作者 Charles Xie 是 Zilliz CEO，Rentong Guo 是 Manu paper 作者——核心 Zilliz 团队投入

## Open Questions

- **Production deployment in Milvus**：§8 future work；当前 Milvus DISKANN 索引仍是原版 DiskANN，Starling 何时进 release 未知
- **GPU + cache optimization**：§8 明示是 next direction；GPU disk graph + block shuffling 形态需重新设计
- **动态数据 (incremental insertion)**：§7 提"static disk index + 动态 in-memory + 周期 merge"模式（与 [Manu §3.5 stream indexing](../concepts/manu-ssd-hierarchical-kmeans.md) 类似）；增量数据如何不破坏 OR(G)？合并触发的 block shuffling 重跑成本如何 amortize？未深入
- **Filter / multi-vector 集成**：Starling 仅 ANNS + RS——与 [topics/attribute-filtering.md](../topics/attribute-filtering.md) (FilteredVamana / ACORN) / [topics/multi-vector-queries.md](../topics/multi-vector-queries.md) 联合？wiki 未覆盖
- **跨 segment 优化**：Starling 单 segment 内优化；多 segment 间 vector 关系（cross-segment NN）未利用——可能造成跨 segment 路由低效
- **Quantizer 升级**：当前用 PQ short codes for routing；替换为 [RaBitQ](../concepts/rabitq.md)（unbiased + sharp error bound）后 routing 决策更准——理论上减少 disk reads 进一步。RaBitQ 与 Starling 同年发表，未实证
- **VBASE iterator + Starling disk layer 叠加**：理论可行（[Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 是 graph-level 性质，与 disk layout 正交）；实证未做
- **Range search 在 segment 多 replica 场景**：vector DBMS segment 通常多 replica；同 query r 在 multiple replicas 上的 result 一致性？论文未涉及
- **OR(G) 是否真的与性能强相关**：Starling §6.3 实测 ξ 与 latency 非线性——其他因素（PQ accuracy, in-mem nav graph quality）也影响。OR(G) 上界研究未深入
- **embedding model 升级**：与 wiki 全 frontier 一致——quantization layer 同样需要重 build

Cited by: 待 query 引用
