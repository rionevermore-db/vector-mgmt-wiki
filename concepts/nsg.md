---
title: NSG（Navigating Spreading-out Graph）
type: concept
sources: [fu-2017-nsg, guo-2019-scann, subramanya-2019-diskann]
related: [hnsw.md, nsw.md, proximity-graph.md, product-quantization.md, scann.md, vamana.md, ../systems/diskann.md, ../topics/mips-vs-l2-nn.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/nsg-vs-graph-anns-million.md]
created: 2026-05-07
updated: 2026-05-07
---

# NSG

**TL;DR**: 单层 proximity graph，靠 MRNG 理论得到常数最大出度 + 高维下接近 O(log N) 的期望搜索复杂度；工程上从近似 kNN graph + 单一 Navigating Node 出发剪边构造，DFS spanning tree 修复连通性。在 SIFT1M / GIST1M / DEEP100M 上系统击败 [HNSW](./hnsw.md)，已部署到 Taobao 十亿级搜索。[fu-2017-nsg §3.5, §4]

## 提出背景

Cong Fu, Chao Xiang, Changxu Wang, Deng Cai（浙江大学），VLDB 2019（arXiv 2017）。

针对当时 graph-ANN 状态的判断：
- [HNSW](./hnsw.md) 是事实最强但有不必要的多层结构开销；
- FANNG / DPG 等 RNG 类没有 monotonicity 保证，搜索可能 detour；
- kNN graph 类（KGraph、Efanna）出度太大、索引太占内存。

作者把所有 graph-ANN 拉到统一的"四个 aspect"标尺下衡量：

1. **图连通性** —— 从 entry point 必须可达任意点
2. **平均出度** —— 影响每跳代价
3. **搜索路径长度** —— 影响总跳数
4. **索引大小** —— 影响 scale

NSG 是首个同时打中四点的算法。[fu-2017-nsg §1, §3.1]

## 三层结构（背景 → 理论 → 工程）

### Layer 1 — MSNET 背景

**Monotonic Search Network**（Dearholt 1988 [13 in fu-2017-nsg]）：图 G 上对任意两点 p, q，存在至少一条单调路径——即朴素贪心搜索（Algorithm 1）无回溯即可到达 q。

- 单调性 ⇒ 强连通性。
- 朴素贪心 = "去当前节点最近邻居中最接近 query 的那个"。
- Delaunay 图是 MSNET，但高维下度数指数膨胀；Dearholt 提出"最小 MSNET"构造但 O(n²log n + n²c) 不可大规模实施。

NSG 论文 §3.2 给出了关键证明：**MSNET 上 Algorithm 1 找到的就是单调路径**（Theorem 1，原 Dearholt 假设但未证），把 MSNET 这一类图变成可分析对象。

### Layer 2 — MRNG（理论核心）

**Monotonic Relative Neighborhood Graph**：在 RNG 基础上改为有向 + 按 index 排序的边选择。

- 边 `pq` ∈ MRNG ⇔ `lune_pq ∩ S = ∅`，或对所有 `r ∈ lune_pq ∩ S` 都有 `pr ∈ MRNG` 且 `index(q) < index(r)`。[fu-2017-nsg Definition 5]
- 关键性质：
  - **NNG ⊂ MRNG** —— 每个节点必然连其最近邻；这是 RNG 不保证的，是 monotonicity 的必要条件（Fig 4 反例）。
  - **最大出度有上界 C_d**，与 n 无关，仅取决于维度 d（Lemma 2）。
  - **期望搜索复杂度** O(c·n^(1/d)·log(n^(1/d))/Δr)，高维下接近 O(log N)。[fu-2017-nsg Theorem 2]
- 致命短板：朴素构造 O(n²log n + n²c)，百万级以上不可行。

### Layer 3 — NSG（工程化近似）

[fu-2017-nsg Algorithm 2]

1. **建近似 kNN graph**：用 nn-descent [14 in fu-2017-nsg] 或 Faiss [28 in fu-2017-nsg]。
2. **找 Navigating Node**：算数据集 centroid，在 kNN graph 上搜其最近邻；这个固定 entry point 是 NSG 与 [HNSW](./hnsw.md) 的关键差异之一。
3. **对每个节点 p 选边**：
   - 从 Navigating Node 在 kNN graph 上 greedy-search 到 p，记下沿路所有访问节点 + p 在 kNN graph 中的邻居 → 候选集；
   - 用 MRNG 边选择策略剪到至多 m 条，丢长边。
4. **DFS spanning tree**：从 Navigating Node 做 DFS，未连通的节点接到其在 kNN graph 中的最近 in-tree 邻居 —— 修复 step 3 截断引入的断连。

经验索引复杂度 O(n^1.16)（远低于 MRNG 的 O(n²log n)），SIFT1M 上 140 + 134 = ~5 min。[fu-2017-nsg §3.5.1, Table 3]

## 关键参数

| 参数 | 典型值 | 作用 |
|---|---|---|
| `k`（kNN graph） | 200–400 | 近似 kNN 图的 k；越大 NSG 越接近 MRNG，索引越慢 |
| `l`（candidate pool） | 100–500 | search-collect 阶段候选池大小 |
| `m`（max out-degree） | 20–50 | NSG 节点最大出度；控制内存 + 准确率 |
| `ef`（query 时） | 100–500 | 搜索时动态候选列表大小 |

[fu-2017-nsg §4.1.4]

## 与 HNSW 的关键差异

| | NSG | [HNSW](./hnsw.md) |
|---|---|---|
| 层数 | 单层 | 多层（指数衰减） |
| Entry point | 单一 Navigating Node（centroid 邻域） | 顶层最高 level 节点 |
| 长程连接来源 | MRNG 边选择策略保留的远邻 | 顶层显式长程边 |
| 连通性保证 | DFS spanning tree | 分层 + 顶层 entry |
| 理论基础 | MRNG / MSNET（论文证明 close-log 复杂度） | RNG 近似（经验 O(log N)） |
| 最大出度 | 常数 C_d（理论） | 经验有界 |
| 内存（SIFT1M） | **153 MB** | 451 MB（仅底层） |
| Million-scale QPS @ 高 precision | **优** | 次 |

[fu-2017-nsg §4.1, Table 2, Fig 6] 详见 [NSG vs Graph ANNs on Million-Scale](../benchmarks/nsg-vs-graph-anns-million.md)。

## 与 PQ / IVFPQ 的对比

- DEEP100M（96-d, 100M）：NSG-16core 比 [Faiss IVFPQ](./product-quantization.md)-1core 在 99% precision 快约 430×，**且击败 Faiss-GPU**。[fu-2017-nsg §4.2 + Fig 7]
- 但 NSG 索引内存 37 GB（DEEP100M 全数据）；HNSW 在同数据上直接 OOM。
- **2B 数据单机不可能** → 必须分布式：32 partition × 12 小时索引 → 5 ms 单查询响应（Taobao 部署）。[fu-2017-nsg §4.3, Table 5]
- 结论：NSG 在数据能装内存时是最快的；PQ 仍是十亿级单机内存下的唯一路径。

## 典型实现

- 作者实现：[`ZJULearning/nsg`](https://github.com/ZJULearning/nsg)（C++）
- Faiss 已支持 NSG 索引（`IndexNSG`）
- 主流向量数据库（Milvus 等）已集成

## Open Questions

- **索引时间**仍是小时级（Taobao 每分区 12 小时），阻碍日级更新；分布式分片是缓解，不是根治。
- **不支持增量更新**（论文 §5 明确承认是未来工作）。
- **2B 单机不可能**——必须依赖 PQ 路径或多机分片。
- **Δr 项**的理论解释不完整——只有经验验证它"近似常数"。
- **Navigating Node 选择**仅用 centroid 邻居；动态数据下 centroid 漂移如何处理未讨论。
- **MIPS 任务下未验证**：MRNG 的 monotonicity 证明依赖 L2 距离结构（lune 几何），MIPS 不是度量空间，理论是否成立未在论文中讨论。NSG 论文全部 benchmark 都是 L2 任务。详见 [topics/mips-vs-l2-nn.md](../topics/mips-vs-l2-nn.md)。
- NSG 隐式 α=1；[Vamana](./vamana.md) 引入可调 α 后**在 SIFT1M / GIST1M / DEEP1M 上系统击败 NSG**（更小的 graph diameter，更少 hops）[subramanya-2019-diskann §4.1]。Vamana 是否完全取代 NSG？争议中。
- NSG 也假设全内存；[DiskANN](../systems/diskann.md) 用 Vamana + SSD 给出"单机 + 大数据"组合，与 Taobao 32-shard 路线不同。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md)
