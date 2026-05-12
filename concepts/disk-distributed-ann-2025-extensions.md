---
title: 2025 Disk + Distributed ANN 三轴扩展 (BatANN / SPI / Gorgeous)
type: concept
sources: [dang-2025-batann, liu-2025-spi, yin-2025-gorgeous]
related: [
  ../concepts/frontier-2025-distributed-vector-search.md,
  ../concepts/distributedann-cited-frontier.md,
  ../systems/diskann.md,
  ../concepts/vamana.md,
  ../concepts/manu-ssd-hierarchical-kmeans.md,
  ../topics/disk-vs-memory-ann.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# 2025 Disk + Distributed ANN 三轴扩展 (BatANN / SPI / Gorgeous)

**TL;DR**: 3 篇 2025-08 ~ 2025-12 arXiv preprint, 在 disk + distributed ANN 上沿**3 条正交轴**做扩展, 与 Ingest #13 (PathFinder / Trinity / SPIRE) 形成互补:
**(1) BatANN** (Dang/Landrum/Birman 2025 Cornell, arXiv 2512.09331) — **inter-node 查询编排**: single global graph + **baton-passing**——访问邻居在其他机器时把 query state 发到目标机执行 (locality first), 10 servers 上 100M-1B vectors **<6ms 均延迟 Recall@10=0.95**, **6.49× / 5.1× throughput** over scatter-gather, **first OSS distributed disk vector search with single global graph**;
**(2) SPI / Semantic Pyramid Indexing** (Liu & Yu 2025, arXiv 2511.16681) — **per-query 分辨率自适应**: VecDB 内多分辨率 hierarchical index + 轻量 classifier 按 query 动态选最佳 resolution level, RAG 不同 semantic granularity 一套系统服务;
**(3) Gorgeous** (Yin et al. 2025, arXiv 2508.15290) — **intra-storage 数据布局**: 关键 insight = **图结构访问 >> 向量访问**, 缓存层优先 graph structure, 60%+ throughput / 35%+ latency reduction vs DiskANN + Starling.

## 提出背景

**为什么这 3 篇 bundle?**
- 都是 **2025-08 ~ 2025-12** 半年窗口期 arXiv preprint, 与 Ingest #13 (PathFinder/Trinity/SPIRE 都集中在 2025-11~12) 同时间窗
- 3 篇都不发明新 ANN 算法, 都在已有 DiskANN/Vamana proximity graph 上做**系统层**优化, 但**各自针对不同瓶颈**:
  - BatANN = **node 间通信瓶颈** (cross-node RPC dominates)
  - SPI = **query 多样性瓶颈** (不同 query 需不同 granularity)
  - Gorgeous = **disk 访问瓶颈** (cache miss pattern 分析)
- 三 paper 与 Ingest #13 SPIRE 形成对比:
  - SPIRE = recursive multi-level + end-to-end accuracy preservation
  - BatANN = single global graph + locality-aware query orchestration (**对立 SPIRE 哲学**)
  - SPI = multi-resolution at VecDB layer (orthogonal to SPIRE 的 index level)
  - Gorgeous = 单节点 disk layout (orthogonal to all 4)

## 三论文 axis 三角

### Axis 1: Inter-node query orchestration — **BatANN (Dang, Landrum & Birman 2025 Cornell)**

[per sources/papers/dang-2025-batann.pdf]

- **问题**: 分布式 disk-based vector search 当前两种方案都有 fundamental 缺陷:
  - **Single global graph + scatter-gather**: 跨 node 访问邻居时, **collect 所有 candidate 返主调 node**, network traffic 大, locality 差
  - **Partition-routing hierarchy** (SPIRE / DSPANN / Pinecone-pod): 减 cross-node 但 **fidelity loss + 多 partition 探测**——SPIRE 已分析
- **关键创新 (Baton-passing)**:
  - 保留 **single global graph** (避免 fidelity loss)
  - 但跨 node 访问时**不 collect 返**, 而是把整个 query state (best-first search queue + visited set + topK) **发到目标 node 继续执行**
  - 类似 distributed graph DB (Pregel / GraphLab) 的 vertex-centric computation, 但应用到 ANN traversal
- **关键 results**:
  - **100M ~ 1B vectors**, 10 servers
  - **<6 ms mean end-to-end latency at Recall@10=0.95**
  - **6.49× (100M)** / **5.1× (1B)** throughput vs scatter-gather baselines
  - Throughput scales **near-linearly** in server count, 同时保留 log-time per-query 搜索效率
- **关键 insight**: cross-node bandwidth cost > query state copy cost——传 query state 反而比传 candidate vector 数据**便宜**, 因为 query state 是 O(log n) entries 而 candidate vector 是 O(K × d) bytes
- **OSS**: 论文承诺 first OSS distributed disk vector search with single global graph

### Axis 2: Per-query resolution adaptivity — **SPI / Semantic Pyramid Indexing (Liu & Yu 2025)**

[per sources/papers/liu-2025-spi.pdf]

- **问题**: 现有 VecDB retrieval pipeline 用 **flat / single-resolution index** (HNSW / IVF / DiskANN 都是单分辨率), 无法适应 RAG 不同 query 的不同 semantic granularity 需求:
  - "What did the CEO say about AI?" → 需要 **document-level** chunks
  - "What's the exact phrase used in section 4?" → 需要 **sentence-level** chunks
  - Single resolution 无法两者都好
- **关键创新 (Semantic Pyramid Indexing, SPI)**:
  - 构造 **semantic pyramid** over document embeddings——多 level chunk granularity (e.g. sentence / paragraph / section / document) 各自 build vector index
  - 用**轻量 classifier** 在 query 时动态决定 best resolution level for that query
  - 不需 offline tuning + 不需 separate model training (与传统 hierarchical 区别)
- **关键 results** (paper 主张): query-adaptive resolution 在 RAG 上比 single-resolution 更好——细节 benchmark 需读 full paper
- **关键 insight**: VecDB 一直把 "embedding granularity" 当 **system-time 决策** (admin 决定 chunk size), 应该当 **query-time 决策** (per-query 动态)
- **关键 wiki 影响**: 与 [Matryoshka](./matryoshka-embedding.md) **不同 axis**——MRL 是同 chunk 不同 dim, SPI 是不同 chunk 同 dim; production 可两者**叠加** (动态 chunk × 动态 dim)

### Axis 3: Intra-storage data layout — **Gorgeous (Yin et al. 2025)**

[per sources/papers/yin-2025-gorgeous.pdf]

- **问题**: 现有 disk-resident proximity graph ANN (DiskANN, Starling) 数据 layout sub-optimal:
  - 内存 cache 同时存 graph structure (neighbor lists) + vectors
  - **未区分**两者的访问 frequency, 简单 LRU/clock 不知 graph structure 远多频访
- **关键创新 (priority caching of graph structure)**:
  - 通过 profiling 实测: **graph structure (邻居 list) 访问 frequency >> vector data 访问 frequency**——proximity graph traversal 每 step 看邻居 list 但只对 frontier 计算 vector distance
  - 设计 Gorgeous **graph-structure-first cache policy**: 内存优先存 graph 邻居数据, vector data 在 SSD on-demand 读
  - 配合 cache-friendly 邻居 list layout (e.g. cluster 邻居共置 disk page) 提升 locality
- **关键 results**:
  - **>60% throughput improvement** over DiskANN + Starling
  - **>35% latency reduction**
- **关键 insight**: "graph structure access ≠ vector data access"——之前所有 disk-resident ANN 都隐含等同, Gorgeous 显式分开是 unique contribution
- **wiki 影响**: 这是 **第 11 类 disk philosophy**——之前 10 类聚焦 "vector 数据放哪", Gorgeous 引入"graph structure vs vector 优先级"作独立 axis

## 三 axis 三论文对比表

| Axis | BatANN | SPI | Gorgeous |
|---|---|---|---|
| 瓶颈层 | Inter-node 网络 | Per-query semantic granularity | Intra-storage cache hit rate |
| 创新性质 | Distributed graph DB pattern → ANN | Multi-resolution + classifier | Profile-driven cache policy |
| 部署规模 | 10 servers, 100M-1B | RAG VecDB single-deployment | 单 node + SSD |
| 相对 SOTA | 6.49× scatter-gather | (vs single-resolution baseline) | 60%+ DiskANN/Starling |
| 关键约束 | "single global graph" 限制——大规模数据 single graph 仍 fit | Classifier accuracy + 多 resolution 索引 storage overhead | SSD I/O + 内存 cache budget |
| 与 SPIRE (#13) 关系 | **对立哲学**——BatANN 保 single graph + 高 locality, SPIRE 弃 single graph + balanced hierarchy | Orthogonal——SPI 在 VecDB 层, SPIRE 在 index 层 | Orthogonal——Gorgeous 单 node 优化, SPIRE 多 node 拓扑 |
| OSS plan | 计划 release | 未明确 | 未明确 |

## 与 Ingest #13 + 已有 disk concept 的关系

```
            分布式 disk-based ANN 设计空间
            ┌────────────────────────────────────────┐
            │  Single Global Graph                    │
            │   ├─ Scatter-gather (传统, 高 RPC)      │
            │   └─ BatANN (baton-passing, 低 RPC)     │ ← Ingest #15 新增
            │                                          │
            │  Partition Routing Hierarchy             │
            │   ├─ DSPANN (per-level optimize)        │
            │   ├─ Pinecone-pod (商用 hierarchical)   │
            │   └─ SPIRE (end-to-end accuracy)        │ ← Ingest #13
            │                                          │
            │  Hardware-specific                       │
            │   ├─ CXL-ANNS (memory disaggregation)   │
            │   ├─ LM-DiskANN (low-memory)            │
            │   └─ AiSAQ (DRAM-free)                  │ ← Ingest #5-8 (DistributedANN-cited)
            └────────────────────────────────────────┘

            单节点 disk-resident ANN 数据布局
            ┌────────────────────────────────────────┐
            │  Vector-centric (default)               │
            │   └─ DiskANN, Starling (vectors > graph)│
            │                                          │
            │  Graph-structure-priority                │
            │   └─ Gorgeous (graph > vectors)         │ ← Ingest #15 新增
            └────────────────────────────────────────┘
```

- BatANN 与 SPIRE 是**对立 design choice**——BatANN: 保 single global graph + 跨 node query state migration, SPIRE: 弃 single global graph + balanced multi-level hierarchy
- 两者代表 **2025 末分布式 disk vector search 的二选一**: 
  - **BatANN 路线**: 保 single graph 高 fidelity, locality 靠 query orchestration 解决
  - **SPIRE 路线**: 拆 hierarchy 减 communication, fidelity loss 靠 end-to-end accuracy 设计补救

## Wiki 影响

### 关键贡献到 wiki 知识图

1. **Baton-passing 作 distributed ANN 新 paradigm** (BatANN): 之前 wiki distributed ANN 描述默认 scatter-gather 或 partition-routing 两类, BatANN 引入第三类——**保 single global graph + query state 跨 node 迁移**. 这是 distributed graph DB (Pregel-style) 思想首次系统应用到 ANN.

2. **VecDB layer multi-resolution adaptivity** (SPI): 之前 wiki "embedding granularity" 视为 admin 设定 (chunk size 在 ingestion 时定), SPI 把它升级为 **runtime per-query 决策**. 与 wiki Matryoshka 维度 axis **正交**, production 可叠加 (动态 chunk + 动态 dim).

3. **Graph structure vs vector data 缓存优先级** (Gorgeous): wiki **首次明确** "graph structure access ≠ vector access" 频率差异. 这是 wiki 第 11 类 disk philosophy axis——之前 10 类聚焦 "vector 在哪" (DRAM/SSD/CXL/etc), Gorgeous 引入 "graph 结构 vs vector 谁优先 in cache" 新独立 axis.

4. **BatANN vs SPIRE 的 2025 末 design opposition**: 两 paper 同窗口期 (2025-12) arXiv, 选择**截然相反**的拓扑哲学. wiki Talk 应明确区分两路线 trade-off.

5. **数据点扩展**:
   - 1B vectors / 10 servers / <6ms / Recall@10=0.95 (BatANN) = **wiki disk-based 单节点扩展的新数据点**
   - 60%+ throughput / 35%+ latency (Gorgeous) = wiki disk-layout 优化天花板感
   - SPIRE 8B / 46 nodes vs BatANN 1B / 10 nodes vs DistributedANN 1.5T / Meta-internal = **三级规模数据点**填空

6. **2025 disk + distributed ANN 6 paper 完整快照**: Ingest #13 (PathFinder/Trinity/SPIRE) + Ingest #15 (BatANN/SPI/Gorgeous) = 6 paper 涵盖 2025 disk/distributed ANN 6 个不同 axis. wiki 已覆盖该年最重要 system-layer 工作.

## Open Questions

- **BatANN single global graph 在万亿规模能否生存**: 1B 是 BatANN paper 验证上限, 万亿规模 single graph 自身需要 sharding——baton-passing 是否仍 viable, 或必须退化到 SPIRE-style hierarchy?
- **SPI classifier accuracy 在 OOD query 上**: SPI 假设 lightweight classifier 能准确选 resolution level. Production OOD query / 多语言 / 多 domain 场景 classifier 失败时 SPI fallback 行为? Paper 未深入.
- **Gorgeous 与 graph quantization (LM-DiskANN per-node PQ) 的 compatibility**: Gorgeous 主张 "graph structure first cache", LM-DiskANN 主张 "per-node 存 PQ codes for neighbor distance"——两者技术目标都是减 disk I/O 但 axis 不同, 是否能 stack? Paper 未交叉.
- **BatANN locality 在 heterogeneous network 上**: paper 假设 10 servers 同 cluster 内, 跨 region / 跨 cloud 网络异质环境下 baton-passing 是否仍胜过 scatter-gather? 实测数据点缺.
- **SPI multi-resolution 与 BGE-M3 multi-functionality 关系**: BGE-M3 同模型输出 sparse/dense/colbert 三 representation, SPI 同 corpus 多 chunk granularity——两者都是 "1-input-multi-output" 但不同 axis, production 是否能合并 design?
- **2025 末 6 paper 都是系统/优化层 incremental**: 没有"破坏性"新算法 (vs HNSW / DiskANN / SPANN 创立期). 是否 ANN algorithm core 已成熟, 后续 5 年都在 system layer 推进? wiki 应跟踪此趋势.

## Cited by

(将随未来 ingest 累积)
