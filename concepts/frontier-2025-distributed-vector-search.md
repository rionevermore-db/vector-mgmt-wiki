---
title: 2025 前沿: 分布式向量检索三轴 (PathFinder / Trinity / SPIRE)
type: concept
sources: [wu-2025-pathfinder, liu-2025-trinity, xu-2025-spire]
related: [
  ../topics/attribute-filtering.md,
  ../concepts/distributedann-cited-frontier.md,
  ../concepts/cagra-graph.md,
  ../concepts/acorn.md,
  ../concepts/filtered-vamana.md,
  ../concepts/manu-ssd-hierarchical-kmeans.md,
  ../concepts/pinecone-pod-based.md,
  ../concepts/pinecone-serverless-slabs.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# 2025 前沿: 分布式向量检索三轴 (PathFinder / Trinity / SPIRE)

**TL;DR**: 3 篇 2025-11 ~ 2025-12 arXiv preprint, 描绘分布式向量检索的 **3 条正交前沿轴**:
**(1) Filter-aware ANNS with cost-based optimizer** — PathFinder (Wu & Tang 2025 UT Austin, PVLDB) 把 RDBMS optimizer 思路引入 vector DB, 处理多属性 conjunction + disjunction 复杂过滤；
**(2) Vector search as 3rd disaggregated tier in LLM serving** — Trinity (Liu & Qian 2025 UCSC) 把 vector search 从 prefill-decode 之外独立 GPU 池, 解决 RAG 尾延迟；
**(3) Accuracy-preserving hierarchical distributed ANN at 8B+ scale** — SPIRE (Xu et al. 2025 USTC + Microsoft Research) **多级 index 端到端精度保留构建**, 8B vectors / 46 nodes / 9.64× SOTA throughput.
本 bundle 填 wiki **2025 vector DB 系统前沿** gap——之前 wiki frontier 截至 DistributedANN (2024 Subramanya) / Gottesbüren (2024); 现 source-back 到 2025-12.

## 提出背景

**为什么这 3 篇 bundle?**
- 三者发布于 2025-11 ~ 2025-12 同一 2 个月窗口期, 是 **2026 PVLDB / SIGMOD 主投递潮**——影响 talk SIGMOD 2026 同侪工作 baseline
- 三者**三种正交 frontier axis**——不是同一问题的不同解, 而是不同问题的同时推进:
  - PathFinder = **filter complexity axis** (单属性 → 多属性 conjunction+disjunction)
  - Trinity = **LLM-system integration axis** (vector search 作 RAG 独立 tier)
  - SPIRE = **distributed scaling axis** (10亿 → 80亿 跨节点, end-to-end 精度)
- 三者共同特征: **不发明新 algorithm**——都基于已有 ANN core (proximity graph / CAGRA / hierarchical clustering), 创新点是 **系统级 architecture + optimizer + scheduler**

## 三论文 frontier 三角

### Frontier 1: Filter-aware ANNS with cost-based optimizer — **PathFinder (Wu & Tang 2025 UT Austin, PVLDB 2026)**

[per sources/papers/wu-2025-pathfinder.pdf]

- **问题**: filter ANNS with **arbitrary conjunctions + disjunctions** over multiple attributes. 现有 attribute-specific 索引 (Filtered-DiskANN, ACORN, iRangeGraph) 仅高效处理 **单属性 filter**, 多属性复杂 filter 退化为 post-filtering 或 in-filtering, 严重降低吞吐
- **架构原则**: 借鉴 RDBMS——"既然为所有 attribute 组合建索引 cost 太高, 让 admin **选择性建 attribute-specific index**, query 时用 **cost-based optimizer 拼接利用**"
- **3 novel 技术**:
  1. **Search utility metric** — 量化 graph density (covers more docs in filter) vs 总 graph 数 (more lookup overhead) 的 trade-off, 不需要昂贵 cardinality estimation, 只需计算 *relative ordering*
  2. **Two-phase optimization for DNF filters** — predicate → disjunctive normal form, 对每个 conjunctive clause 选 top-2 promising attribute-specific 索引 subset, 然后跨 clause 选 highest-utility 集合; novel 算法 from tree-based attribute index 找 **common ancestor proximity graph 可 subsume + 替代 group 内全部 graph**
  3. **Index borrowing** — 用户对某 attribute 没建索引时, 利用 attribute 间相关性 **synthesize 新 predicate on indexed attribute**, 重用 attribute-specific index 处理 unindexed filter
- **关键结果**: 4 datasets, **9.8× higher query throughput at recall 0.95** vs best baseline. 仅毫秒级 query budget 内做 optimizer 决策 (sub-ms ~ few-ms search 内必须完成 optimizer 选择)
- **vision quote**: "PathFinder opens a new direction for supporting filtered ANNS in vector databases by drawing on successful practice from relational databases: adopting a cost-based optimizer to best utilize available attribute-specific indexes"

### Frontier 2: Vector search as 3rd disaggregated tier — **Trinity (Liu & Qian 2025 UCSC)**

[per sources/papers/liu-2025-trinity.pdf]

- **问题**: PD (Prefill-Decode) disaggregation 是 industry LLM serving 主流——prefill 用大 GPU 池 compute-bound (大 GEMM); decode 用单独池 memory-bound (KV-cache); 但 **vector search for RAG 仍与 model inference 纠缠 in same GPU**, 导致 tail latency 膨胀
- **核心 insight (roofline analysis)**: 向量检索 (CAGRA-style GPU 图遍历) 与 decode 同为 **memory-bound** 但 batch size 最优点不同——prefill saturate at 100% (compute-bound)、decode plateau low (BW-bound)、vector search plateau at BW roof. **三阶段最优 batch size 不同 → 应跑在独立 GPU**
- **3 architecture 备选 + 决策**:
  - (a) Coupled: vector GPU 与 LLM GPU 同 server, NVLink shortest latency, 但 contend GPU 资源 + 替换 EP expert GPU 引入 dispatch/combine 跨节点流量, 抵消收益
  - (b) Partially-coupled: vector GPU 与 prefill 同 server (decode 拉远), 适合 prefill 主导 RAG, 但 EP traffic 仍跨节点
  - (c) **Decoupled (Trinity)**: vector search 完全独立 GPU 池, prefill + decode 不预留 GPU 给 vector, 仅跨网络 RPC——**胜出**——避免 EP traffic 干扰, retrieval 池 ~ document store 通信路径独立
- **3 innovations**:
  1. **架构选 (c) decoupled**——本 paper 首个正式分析 PD + vector search 3 种 deployment 拓扑
  2. **Continuous batching for vector search on GPU** — fixed-degree graph (CAGRA) in HBM + per-request topM/visited table device-side, scheduler loop 每 *extend* step 选 up-to-p non-expanded parents, 让不同 request 在不同 extend depth 共用 batch (类似 LLM continuous batching for diff. seq positions)
  3. **Stage-aware scheduling + preemption** — multi-priority queues balance TTFT (time-to-first-token, prefill 期 retrieval 高优先级) vs throughput (decode 期 retrieval 可暂停)
- **关键结果**: independent vector search pool 在 PD-disaggregated LLM serving 中维持 high throughput + low tail latency for 异质 RAG workloads
- **vision quote**: 这是 wiki 内 **首篇正式 "vector search ↔ LLM serving system orchestration" paper**——vector search 不只是数据库, 更是 LLM inference pipeline 的 3rd tier

### Frontier 3: Accuracy-preserving hierarchical distributed ANN at 8B+ — **SPIRE (Xu et al. 2025 USTC + Microsoft Research)**

[per sources/papers/xu-2025-spire.pdf]

- **问题**: 分布式 ANN scaling 到 80 亿向量, 两种主流路径都有结构问题:
  - **Naive sharding** of dense graph (HNSW): 实测 **>80% search steps 是 cross-node**——dense connectivity 跨 shard 引入大量 RPC, p99 latency 上升 2 个数量级
  - **Partition-routing hierarchy** (DSPANN / SPTAG / Pinecone pod / KD-tree / R-tree): 减 cross-node communication, 但 **partition boundary 引入 fidelity loss**——query 接近 boundary 时被错路由, 为补救必须 probe 多 partition (DSPANN 8B index 中 9/46 partition 探测才 reach recall@5=0.9), 严重打掉 throughput
- **2 设计决策**:
  1. **Balanced partition granularity**——量化 partition density 与 (a) avg vector read cost (b) cross-node steps 的关系, 找 **inflection point**: 不能太 dense (read amplification 严重) 也不能太 sparse (cross-node 多), 中间存在 sweet spot
  2. **Accuracy-preserving recursive index construction**——与传统 hierarchical (DSPANN / ADBV) 关键区别: 传统 **每层独立优化** level accuracy, 但 fidelity loss 跨层累积; SPIRE **每层构造时优化 end-to-end accuracy**, 把 fidelity budget 整体分配
- **architecture**:
  - Root (top) level: in-memory proximity graph
  - 下层: vector clusters 存 SSD
  - **Bottom-up 构造**: cluster 所有 vectors at chosen granularity → recursively cluster centroids 直到 root fit single-server memory
  - 每层 query 选 top-m partitions 平行并行 (1 个 network round-trip)
  - **Stateless compute tier**: in-memory root replicated, SSD level easy reconstruct on failure
- **关键结果**: 8B vectors / 46 nodes / **9.64× peak throughput** vs SOTA, lower avg + tail latency at all scales. Saturate SSD I/O while only using <30% network + <40% CPU——**仍有 headroom for further scaling**
- **codebase**: SPIRE 实现 ~6000 行 C++, 论文承诺 release
- **vision quote**: "This recursive, level-by-level construction provides fixed search cost per level, resulting in stable and highly predictable end-to-end search behavior across scales"——**predictability 比 absolute peak performance 更重要** for production

## 三 frontier paradigm 对比表

| Axis | PathFinder | Trinity | SPIRE |
|---|---|---|---|
| 主要问题 | Filter complexity (多属性 conjunction + disjunction) | LLM serving 中 vector search orchestration | Distributed scaling at 8B+ vectors |
| 创新层次 | Query optimizer (RDBMS-style cost-based) | System architecture (PD + vector 3-way disagg.) | Index construction (multi-level end-to-end accuracy) |
| 不发明 algorithm 的程度 | 全部基于已有 attribute-specific index | 基于 CAGRA (已有 GPU graph ANN) | 基于已有 IVF / proximity graph cluster |
| 关键 metric 提升 | 9.8× throughput at recall 0.95 | 维持 SLO under heterogeneous RAG | 9.64× peak throughput at 8B |
| 部署规模 | 单节点 (admin 建多 attribute-specific 索引) | GPU 集群 (PD pool + vector pool) | 46 节点 (8B vectors) |
| 直接 Talk relevance | 弱 (filter axis 与千亿 sharding 正交) | 中 (LLM serving 与 vector mgmt 桥接) | **强** (8B / 46 节点直接是 Talk 主题问题域) |
| Open source | 未明确 | 未明确 | 计划 release codebase |

## 与同类 / 前作的对比

| 与本 bundle 对比 | 共同点 | 差异 |
|---|---|---|
| [Attribute Filtering 五策略](../topics/attribute-filtering.md) | filter-vector 联合 query | 五策略框架 = filter execution **strategy** (pre / post / 双索引 / filter-aware graph / inline); PathFinder = filter strategy 之上的 **optimizer**——选哪种 strategy + 哪个 index, 更高一层 |
| [ACORN](./acorn.md) | filter-aware graph | ACORN = neural attribute filter via prompt, single attribute; PathFinder = multi-attribute + conjunction/disjunction, **不发明 attribute-specific index 本身**, 而是组合现有索引 |
| [Filtered-Vamana](./filtered-vamana.md) | filter-aware graph | 同上, Filtered-DiskANN 是 single attribute; PathFinder 利用 + 组合多个 Filtered-DiskANN |
| [DistributedANN bundled](./distributedann-cited-frontier.md) | distributed scaling | DistributedANN 系列 = 不同 hardware 方向 (CXL / DRAM-free / 多数据集); SPIRE = 经典 partition-routing **改进**——end-to-end 精度构造而非 per-level |
| [Manu SSD hierarchical k-means](./manu-ssd-hierarchical-kmeans.md) | hierarchical SSD-based | Manu (ZILLIZ Milvus 2.0) 是 **production 2 级** hierarchical; SPIRE 是 **recursive multi-level** + 端到端精度优化, 更通用 |
| [Pinecone pod / serverless](./pinecone-pod-based.md) [(serverless)](./pinecone-serverless-slabs.md) | 商业 hierarchical | Pinecone 是闭源商用版 hierarchical (pod 和 slab 两代); SPIRE 是 OSS 学术系统 + 直接 benchmark 与之比较 |
| [CAGRA Graph](./cagra-graph.md) | GPU-based ANN | CAGRA 是 algorithm (GPU graph 构造); Trinity 把 CAGRA 部署在 PD-disaggregated LLM serving 中, 是 **system 层包装**, 不修改 CAGRA 本身 |

## Wiki 影响

### 关键贡献到 wiki 知识图

1. **首个 ANNS cost-based optimizer concept** (PathFinder) — 之前 wiki vector DB 在"query 时如何选择 index/strategy"是隐式的, PathFinder 引入 **explicit cost-based optimizer** with utility metric + DNF rewriting. 影响 `topics/attribute-filtering.md` 五策略框架的 future 升级——从 "5 个 strategy 二选一" 到 "optimizer 选 strategy + 索引子集".

2. **首个 vector search 作 LLM serving 独立 tier concept** (Trinity) — 之前 wiki 提到 RAG 但 vector search 仅是 "RAG 的 retrieval 部分", **没有 RAG-side system orchestration concept**. Trinity 引入 prefill/decode/vector 3-tier disaggregation, 这是 **2026+ AI 系统设计基础概念**, 对 wiki Talk SIGMOD 2026 中的 "vector mgmt for LLM inference" 主题直接相关——vector mgmt **不只是 DB**, 是 **LLM serving stack 的 3rd tier**.

3. **End-to-end accuracy preservation 作为 hierarchical index 构造原则** (SPIRE) — 之前 wiki 描述 hierarchical ANN (DSPANN / SPTAG / Pinecone pod) 都默认 **per-level 优化**, SPIRE 显式指出"**per-level optimal ≠ end-to-end optimal**", 引入 "fidelity budget 跨层分配"概念. 这影响 wiki `concepts/manu-ssd-hierarchical-kmeans.md` 和 `topics/disk-vs-memory-ann.md` 描述的 hierarchical 系统设计原则.

4. **8B 向量 / 46 节点真实生产数据点**: SPIRE 实测数字直接 instrument `topics/giga-scale-sharding`-style query——之前 wiki 该规模数据点最大是 DistributedANN (Meta 1.5T vectors at fb-internal scale)、Manu (千亿). SPIRE 8B / 46 节点是 OSS 学术系统 + 详细 benchmark 公开, 是 talk demo 可信引用源.

5. **新 concept 增加**: cost-based ANNS optimizer / continuous batching for vector search / PD disaggregation / end-to-end accuracy preservation —— 4 个 production 重要工程概念 wiki 内首次出现.

6. **3 paper 涵盖 4 类近期 frontier 共同特征**: (a) 都基于已有 algorithm core, 创新点在 system layer; (b) 都给 ~10× 改进 数量级 (9.8× / SLO维持 / 9.64×); (c) 都明确指出 wiki 内未充分覆盖的 baseline gap (RDBMS optimizer / LLM serving disaggregation / hierarchical fidelity); (d) 都在 2025-11 ~ 2025-12 同窗口期发布——**2025 末是 distributed vector search 系统层创新潮**.

### 三 paper 在 production stack 的位置

```
                  [LLM Serving Pool]
                  ├─ Prefill GPUs (compute-bound)
                  ├─ Decode GPUs (memory-bound)
                  └─ Vector Search GPUs (Trinity 创新点) ─→
                                                          │
                  [Vector DB Tier]                       │
                  ├─ Filter Layer (PathFinder optimizer ←┘)
                  ├─ Index Layer (SPIRE multi-level)
                  └─ Storage Layer (SSD + in-memory root)
```

- Trinity 在 **serving stack 顶层**: vector search 是 LLM inference 的 retrieval primitive
- PathFinder 在 **filter + index 之间**: query 来后, optimizer 决定 strategy + index subset
- SPIRE 在 **index 内部**: multi-level hierarchy + storage tier 设计

## Open Questions

- **PathFinder optimizer 在 streaming / freshness 场景**: 增量更新 attribute-specific 索引时, optimizer 的 utility metric 是否还稳定? Paper 没涵盖 dynamic + updates.
- **Trinity 在边缘部署 / serverless**: 完全 decoupled vector search GPU 池要求 always-on 大资源, edge / serverless 场景是否退化为 coupled (a)? Trinity 未触及 cost economics.
- **SPIRE 与 Trinity 是否兼容**: SPIRE 假设 CPU + SSD storage tier, Trinity 假设 GPU + HBM——理论上 SPIRE 的 root level (in-memory proximity graph) 可放 GPU HBM 作 Trinity 输入, 但 paper 未交叉验证.
- **PathFinder + SPIRE 联合**: filter-aware optimizer 在 multi-level hierarchical index 中如何选 partition? Paper 都假设 single-level + single-node, **分布式 filter ANNS with cost optimizer 仍是 open**.
- **2026 SIGMOD live demo 主题与本 bundle 关系**: 跨 embedding model 迁移 (Talk 主题) 与本 bundle 三 axis 都未直接交集——这是 talk 论文的 contribution gap, 也是为何这是 Talk 而非 incremental work.
- **8B → 100B → 1T 是否 SPIRE recursive 自然支持**: SPIRE root 内存放约束 (top-level proximity graph fit single server) 决定 max scale. 1T 时 root 自身需要 sharding, paper 未涵盖.

## Cited by

(将随未来 ingest 累积)
