---
title: HNSW（分层可导航小世界图）
type: concept
sources: [malkov-2016-hnsw, fu-2017-nsg, guo-2019-scann, douze-2024-faiss-library, subramanya-2019-diskann, zhang-2023-vbase, wang-2024-starling, singh-2021-freshdiskann, ootomo-2023-cagra]
related: [nsw.md, proximity-graph.md, product-quantization.md, nsg.md, scann.md, vamana.md, acorn.md, relaxed-monotonicity.md, block-shuffling.md, freshvamana.md, cagra-graph.md, ../systems/faiss.md, ../systems/diskann.md, ../systems/milvus.md, ../systems/pase.md, ../systems/vbase.md, ../systems/starling.md, ../systems/freshdiskann.md, ../systems/cagra.md, ../topics/mips-vs-l2-nn.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../topics/attribute-filtering.md, ../topics/topk-vs-iterator-model.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/gpu-vs-cpu-ann.md, ../benchmarks/hnsw-vs-faiss-200m-sift.md, ../benchmarks/nsg-vs-graph-anns-million.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/pase-vs-cube-freddy.md, ../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md, ../benchmarks/vbase-8queries-recipe1m.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md, ../benchmarks/freshdiskann-streaming-sift800m.md, ../benchmarks/cagra-vs-hnsw-ggnn-ganns.md]
created: 2026-04-30
updated: 2026-05-09 (CAGRA)
---

# HNSW

**TL;DR**: 把 NSW 的 proximity graph 按 skip-list 的方式分层：顶层稀疏长程边、底层包含全部元素，搜索从顶层贪心下降，得到 O(log N) 的 ANN 查询与插入复杂度。[malkov-2016-hnsw §3]

## 提出背景

Yu. A. Malkov 与 D. A. Yashunin（2016 arXiv 预印，2020 IEEE TPAMI / Information Systems 正式发表）。针对前作 [NSW](./nsw.md) 的两个痛点：

1. NSW 单次贪心搜索是 polylog 复杂度——zoom-out 阶段平均跳数与节点度数都随网络对数增长，乘起来变成 polylog。[malkov-2016-hnsw §3]
2. NSW 在聚类 / 低维数据上严重退化，可能比基于树的算法慢几个数量级 [34 in malkov-2016-hnsw]。

## 关键性质

- **复杂度**：搜索与插入均为 O(log N)，证明依赖 Delaunay 图度数有界假设——低维 Euclidean 成立，高维只有经验证据。[malkov-2016-hnsw §4.2.1]
- **层级生成**：每个元素 level `l = ⌊−ln(unif(0,1))·m_L⌋`，几何分布，与 skip-list 同构。[malkov-2016-hnsw §3, §4.1]
- **内存开销**：约 `(M_max0 + m_L·M_max)·bytes_per_link`，典型 60–450 字节/对象，不含原始向量数据。[malkov-2016-hnsw §4.2.3]
- **构建可并行**：插入仅有少量同步点，对索引质量无可测影响。[malkov-2016-hnsw §4.2 + Fig 9]

## 三个核心机制

### 1. 层级化（与 NSW 的关键差异）

- 多层 [proximity graph](./proximity-graph.md)，顶层只含少量元素 + 长程链接，底层（Layer 0）包含全部数据。
- 搜索从固定顶层 entry point 开始，每层贪心走到局部最优后下降一层，直到 Layer 0。[malkov-2016-hnsw §3 + Fig 1]
- 与 [NSW](./nsw.md) 的对比：NSW 单层图、靠 hub 节点提供长程连接；HNSW 把"长短链接"显式拆到不同层。

### 2. 启发式邻居选择（Algorithm 4）

- 不简单选 M 个最近候选，而是只在"候选 c 比已选邻居中任一个都更接近插入元素 q"时才接受。[malkov-2016-hnsw Alg 4 + Fig 2]
- 效果：保留跨簇边，得到 relative neighborhood graph（RNG）的近似——一个 Delaunay 图的最小子图。[46 in malkov-2016-hnsw]
- 没有这个启发式，HNSW 在簇间会断连，搜索卡在簇边界。Alg 3（朴素选 M 个最近）是消融基线。[malkov-2016-hnsw §5.2 Fig 7]

### 3. 顶层固定 entry point

- 干掉了 NSW 的 polylog "zoom-out"（从随机低度节点出发，逐步爬到高度 hub）。
- 代价：失去 NSW 的天然分布式特性——无法把图拆到独立节点上做并行搜索。[malkov-2016-hnsw §6]

## 关键参数

| 参数 | 典型值 | 作用 |
|---|---|---|
| `M` | 6–48 | 每层每元素连接数；越大越准也越占内存 |
| `M_max0` | 2·M | 底层连接数上限（独立于其他层） |
| `m_L` | 1/ln(M) | 层级分配的归一化常数；最优值使层间重叠 ≈ 1/M |
| `efConstruction` | 100–500 | 构建时动态候选列表大小；≥100 即可达 ≥0.95 召回 |
| `ef` | 查询时调 | 查询时候选列表大小；recall / 速度调节钮 |

[malkov-2016-hnsw §4.1, Fig 3-8]

## 与同类对比

| | HNSW | NSW | [NSG](./nsg.md) | [Faiss IVFPQ](./product-quantization.md) |
|---|---|---|---|---|
| 数据结构 | 多层 proximity graph | 单层 proximity graph | 单层 + MRNG 近似 | 倒排表 + PQ 量化 |
| 搜索复杂度 | O(log N)（经验） | polylog | ≈ O(log N)（MRNG 理论） | 近似 O(√N) 桶 |
| 内存 | 64 GB（200M SIFT） | 同级 HNSW | **153 MB（SIFT1M）** [fu-2017-nsg Table 2] | 23–30 GB（200M SIFT） |
| Million-scale 速度 @ 高 precision | 次 | 弱 | **最强** [fu-2017-nsg Fig 6] | 弱 |
| 支持删除 / 更新 | 否 | 否 | 否 | 是 |
| 分布式 | 困难 | 天然 | 困难（Taobao 用分片） | 是 |

[malkov-2016-hnsw §5.4 + Table 3 + Fig 15]；NSG 一侧详见 [NSG vs Graph ANNs on Million-Scale](../benchmarks/nsg-vs-graph-anns-million.md)；HNSW vs Faiss 一侧详见 [HNSW vs Faiss PQ on 200M SIFT](../benchmarks/hnsw-vs-faiss-200m-sift.md)。

## 典型实现

- 作者实现：[`nmslib/hnsw`](https://github.com/nmslib/hnsw)，C++ header-only，支持增量构建。
- **HNSWlib** 后来成为 HNSW 的事实参考实现，被 Faiss、[Milvus](../systems/milvus.md) 等主流库引用 [55 in douze-2024-faiss-library]。
- **[Milvus](../systems/milvus.md)** 在 graph-based 索引里直接集成 HNSW 与 RNSG（[NSG](./nsg.md) 变体），与 quantization-based 索引（IVF_FLAT / IVF_SQ8 / IVF_PQ）并列 [wang-2021-milvus §2.2]。Milvus_HNSW 在 SIFT10M / Deep10M 上比 Vearch / 商业系统 A/C 快 7×-73× [per benchmarks/milvus-vs-prior-sift10m-deep10m.md]。
- **[PASE](../systems/pase.md)** 在 PostgreSQL 内核中集成 HNSW 作为 PG 的 first-class index type [yang-2020-pase §2.2]。8KB page-aligned 实现：meta-page + data-page + neighbor-page 三类。但 build 时间显著比 IVFFlat 慢 (SIFT 1M 19×, GIST 1M 56×) ——PG 内核单线程约束。
- **[ACORN](./acorn.md)** [patel-2024-acorn] 是 HNSW 的 **predicate-agnostic 改造**：构造时每 node 收 M·γ 而非 M candidate edges，pruning 用 predicate-agnostic compression（保留 M_β + truncate）。Search 时 GET-NEIGHBORS 加 predicate filter（ACORN-γ 简单 filter + truncate；ACORN-1 + 2-hop expansion）。**让 HNSW 支持 unbounded predicate set + 任意 operator**——25M LAION 实测 >1000× over HNSW post-filter at 0.9 recall。详见 [concepts/acorn.md](./acorn.md)。
- [Faiss `IndexHNSW`](../systems/faiss.md)（Facebook Research）自 2018 年起内置 HNSW 实现，支持与 IVF（`IndexHNSWFlat`）、ScalarQuantizer（`IndexHNSWScalarQuantizer`）等组合。在 IVF 索引中也常被用作 **HNSW-as-coarse-quantizer**（`IVF_HNSW`）。[douze-2024-faiss-library §5.1, §A.7]
- 工程要点：避免使用通用 BLAS 距离函数；C 风格手动内存管理 + prefetch，比 nmslib 通用框架显著更快。[malkov-2016-hnsw §5]

## Open Questions

- 高维下 Delaunay degree 是否真的有界？只有经验验证，无理论证明。[malkov-2016-hnsw §4.2]
- M 参数能否通过启发式自动推断？作者说"潜在可以"但未实现。[malkov-2016-hnsw §6]
- 如何支持元素删除 / 更新而不退化图质量？开放问题。
- 如何让 HNSW 真正分布式？只能简单分片（每分片一个独立 HNSW），全系统吞吐 scale 差。[malkov-2016-hnsw §6]
- [NSG](./nsg.md) 在 million-scale 上系统击败 HNSW（更小内存、更高 QPS）[fu-2017-nsg §4.1]；HNSW 的多层结构在数据能装内存的场景下是否仍是最优？开放争议。
- MIPS 任务下 [ScaNN](./scann.md) 在 Glove1.2M 高 recall 区间击败 HNSW（包括 nmslib 与 faiss 实现）[guo-2019-scann §5.3 + Fig 4b]；HNSW 的 L2 中心设计是否适合 MIPS 主导的现代 embedding 检索场景？详见 [topics/mips-vs-l2-nn.md](../topics/mips-vs-l2-nn.md)。
- HNSW 的边选择启发式（Alg 4）隐式使用 α=1；[Vamana](./vamana.md) 把 α 暴露为可调参数后击败 HNSW（million-scale）[subramanya-2019-diskann §2.4 + §4.1]。HNSW 的多层 hierarchy 是否仍是必要的？开放争议。
- HNSW 假设 index 全在 DRAM；当数据规模超过单机 DRAM 时只能简单分片。[DiskANN](../systems/diskann.md) 给出 SSD-resident 替代路线（单机 64 GB RAM 跑 1B SIFT @ 98% recall）。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。
- HNSW 的 zoom-in / zoom-out 两阶段搜索是 [Relaxed Monotonicity](./relaxed-monotonicity.md) 的特例——zoom-in 是 Phase 1（接近 query），zoom-out 是 Phase 2（离开 query）。但 [malkov-2016] 论文未把这个性质 expose 为 query engine 可用的接口；[VBASE](../systems/vbase.md) [zhang-2023] 第一次形式化为 RM 并加 `amisrm()` iterator API。这让 HNSW 在 vector + relational query engine 中可以摆脱 TopK black-box 角色，作为 first-class iterator 与 B-tree 并列。
- **HNSW 在 disk-resident segment 场景的延伸**：[wang-2024-starling §6.7] 提出 **Starling-HNSW** ——把 HNSW 的 upper layers 当 in-memory navigation graph + layer-0 走 disk + [block shuffling](./block-shuffling.md)。实测 BIGANN 33M 比 Disk-HNSW baseline **2× 快**。这是 HNSW 多层结构在 disk-resident 场景的天然适配——upper layers 已经是"sampled top-level navigation"。详见 [systems/starling.md](../systems/starling.md)。
- **HNSW streaming insert/delete 失败的根因（NEW from FreshDiskANN）**：[singh-2021-freshdiskann §3.3 + Fig 1] 实测 HNSW 在 50 cycles × 5% delete+re-insert SIFT1M 后 recall 95% → 90%；两种 natural delete policy (A 与 B) **都失败**。**根因**：HNSW 的 RobustPrune 是隐式 α=1（aggressive pruning）→ graph 极稀疏 → 删点失去 navigability。FreshDiskANN 证明只有 **α > 1 RobustPrune (Vamana α=1.2)** 才能维持 streaming recall。理论上 HNSW 加入 α-augmented RobustPrune 也可解（**logical community PR**），但 hnswlib / nmslib / Faiss 当前实现没改。详见 [concepts/freshvamana.md](./freshvamana.md) 与 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)。
- **HNSW 在 GPU 上的等效**：[CAGRA](./cagra-graph.md) [ootomo-2023] 是 NVIDIA 提出的 GPU-native graph——**fixed out-degree + non-hierarchical**（替代 HNSW 多层结构）；HNSW 多层是 CPU greedy descent 优化，GPU 高并行度可以 random sampling 替代。CAGRA build 比 HNSW (CPU 64-core) **2.2-27× 快**, large-batch search **33-77× 快**——CPU graph 与 GPU graph 是不同 hardware path 的对偶。详见 [systems/cagra.md](../systems/cagra.md) 与 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md)。

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md)
