---
title: FreshVamana（首个支持 streaming insert/delete 的 graph ANN 算法）
type: concept
sources: [singh-2021-freshdiskann]
related: [vamana.md, hnsw.md, nsg.md, filtered-vamana.md, block-shuffling.md, lire.md, proximity-graph.md, ../systems/diskann.md, ../systems/freshdiskann.md, ../systems/spfresh.md, ../systems/milvus.md, ../topics/in-place-vs-out-of-place-updates.md, ../benchmarks/freshdiskann-streaming-sift800m.md]
created: 2026-05-09
updated: 2026-05-09
---

# FreshVamana

**TL;DR**: Microsoft Research 在 2021 提出的 [Vamana](./vamana.md) 增量变体——**wiki 内首个 graph-based ANNS 算法支持 streaming insert + delete 且 recall 不退化**。核心 insight：[Vamana](./vamana.md) 论文 [subramanya-2019] 引入的 **α-RNG property（α > 1 RobustPrune）不仅是性能优化，更是 fresh-ANNS 必要条件**——HNSW / NSG / Vamana(α=1) 的 aggressive pruning 创建过稀疏 graph，删点后失去 navigability，recall 持续下降；α > 1 的 relaxed pruning 保证 graph 在 update 流下保持 navigability。FreshVamana 是 [FreshDiskANN](../systems/freshdiskann.md) 系统的算法核心，也是 [SPFresh](../systems/spfresh.md) cluster-path 路径的 graph 路径"姐妹工作"。**实测 50 cycles 删除+插入 5%/10%/50% 后 recall 稳定 95%+**（α=1 同条件下从 95% → 90%）。Build 速度也比 static Vamana **1.48-1.83× 快**——成为 default Vamana build 算法的 strict superset。

## 提出背景

[singh-2021-freshdiskann §1, §3.3]

工业 ANN 部署面对 **streaming corpus**（document index、email server、enterprise search 等）需要：
- (a) 实时反映 inserts/deletes 到 index
- (b) 不损失 search performance / recall
- (c) **single machine** scalability（不是 25 台机器跑 PLSH）

但**所有已知 graph-based 算法都是 static**（HNSW [malkov-2016] / NSG [fu-2017] / Vamana [subramanya-2019]）；工业实践只能"周期 rebuild"——48-core 机器 100M HNSW 需 1.5-2 小时；billion-scale 需多机 + 几小时。

### Naive delete policies 的失败（§3.3 + Fig 1）

[singh-2021-freshdiskann §3.3]

实测 SIFT1M 上 HNSW / NSG / Vamana(α=1) 的两种自然 delete policy 都失败：

**Delete Policy A**：删 p 时只移除 p 的 in/out edges。
**Delete Policy B**：删 p 时移除 + 对 p 的每对 (in-neighbor p_in, out-neighbor p_out) 添加补偿边 (p_in, p_out) + RobustPrune。

```
Effect on SIFT1M, 20 cycles × 5% delete + 5% insert:

Delete Policy A:
  HNSW recall: 95% → 90% (持续下降)
  Vamana(α=1) recall: 95% → 89%
  NSG recall: 95% → 88%

Delete Policy B (with α=1 RobustPrune):
  Same downward trends — α=1 仍失败
```

→ **问题不在 policy，而在 RobustPrune 的 α 参数**。

## α-RNG Property —— FreshVamana 的关键洞察

[singh-2021-freshdiskann §4 + Fig 3]

[Vamana](./vamana.md) 论文引入的 α-RNG（α > 1 relaxed Relative Neighborhood Graph）：

```
RobustPrune retains edge (p, p'') only if:
  ∄ edge (p, p') with p' significantly closer to p'' than p, i.e.,
  d(p', p'') < d(p, p'')/α

α = 1: 严格 RNG—每条 edge 都被尽可能 prune
α > 1: relaxed RNG—保留更多"detour edges"
```

**FreshVamana 的发现**：α > 1 不仅在 build 时让 graph 更稠密（DiskANN paper 已知），**更在 update 流下保证 graph 维持 navigability**：

[Fig 3: SIFT1M Deep1M 50 cycles × 5% delete+insert]

| α | 50 cycles 后 recall |
|---|---|
| **1** (HNSW/NSG default) | **95% → 90% (-5%)** |
| 1.1 | 95% → 94% |
| **1.2** (FreshVamana **default**) | **95% 稳定** |
| 1.3 | 95% 稳定（稍慢） |

→ **α=1.2 是 fresh-ANNS 的默认配置**——比 static Vamana α=1.2 多了"recall stability"价值。

> **wiki 解读**：α 参数从"Vamana 性能调节钮"升级为 **fresh-ANNS 必要条件**。这是 graph ANN 文献的一个核心 conceptual shift——之前论文（[subramanya-2019]）只把 α 当 build-time tuning param。

## Insert 算法（Algorithm 2）

[singh-2021-freshdiskann §4.1 Algorithm 2]

```
Insert(x_p, s, L, α, R):
  V ← ∅                                    // 已访问点集
  L ← ∅                                    // 候选列表
  [L, V] ← GreedySearch(s, x_p, 1, L)      // 当前 graph 上跑 search
  N_out(p) ← RobustPrune(p, V, α, R)        // 用 visited set 选 out-neighbors

  // bi-directional edge update + degree control
  for j in N_out(p):
    if |N_out(j) ∪ {p}| > R:
      N_out(j) ← RobustPrune(j, N_out(j) ∪ {p}, α, R)
    else:
      N_out(j) ← N_out(j) ∪ {p}
```

**fine-grained locking**：每个 N_out(p) 是 RW-lock；insert/delete 锁单 vertex 而非全 graph → multi-thread insert throughput 近线性扩展（Appendix）。

**Insertion latency**：~1 ms per insert（mean）on SIFT1B 32GB DRAM 实测。

## Delete 算法（Algorithm 4）—— Lazy + Batch Consolidation

[singh-2021-freshdiskann §4.2 Algorithm 4]

**问题**：Eagerly deletion (即时编辑 graph) 需要更新 p 的所有 in-neighbors——R_in 可能很大（最坏全 graph）→ 不可行。

**FreshVamana 的解决**：

```
Lazy delete:
  delete request → add p to DeleteList; graph unchanged

Search-time:
  use DeleteList to filter out deleted points from result

Batch consolidation (when DeleteList grows to 1-10% of |P|):
  for each p in P\D s.t. N_out(p) ∩ D ≠ ∅:
    D' = N_out(p) ∩ D            // p 的被删 out-neighbors
    C = N_out(p) \ D             // p 的非被删 out-neighbors
    for each v in D':
      C = C ∪ N_out(v)            // 加被删邻居的邻居（补偿 navigability）
    C = C \ D
    N_out(p) = RobustPrune(p, C, α, R)
```

**复杂度**：
- Delete request: O(1)（仅 DeleteList append）
- Consolidation: O(|D|·R²) expected over random delete set——linear in delete set size
- Search: O((1 + |DeleteList|/|P|) × normal cost)——只要 DeleteList <10% 就 negligible

## Recall stability（§4.3 + Fig 2）

[singh-2021-freshdiskann §4.3]

50 cycles × {5%, 10%, 50%} delete+insert on SIFT1M / DEEP1M / GIST1M / SIFT100M：

| Cycle 数 | 5% change recall | 10% change recall | 50% change recall |
|---|---|---|---|
| 0 (initial) | 95% | 95% | 95% |
| 50 | **95%** | **95%** | **95%** |

**4 个 dataset × 3 个 change rate = 12 个实验全部 stable**——证明 α=1.2 在不同 dataset / 不同 change rate 下都 work。

## Build 速度优势（§B + Table 1）

| Dataset | Static Vamana | **FreshVamana** | Speedup |
|---|---|---|---|
| SIFT1M | 32.3 s | 21.8 s | **1.48×** |
| DEEP1M | 26.9 s | 17.7 s | 1.52× |
| GIST1M | 417.2 s | 228.1 s | **1.83×** |
| SIFT100M | 7187.1 s | 4672.1 s | 1.54× |

→ **FreshVamana 是 strict superset of Vamana**——build 更快 + 支持 streaming + 同等 recall。逻辑上应替代 static Vamana 作 default Vamana 算法（DiskANN 当前仍用 static Vamana 是历史遗留）。

## 与同类算法对比

| | HNSW（static） | NSG（static） | Vamana α=1 | **FreshVamana α=1.2** |
|---|---|---|---|---|
| 增量 insert | 支持但 recall 退化 | ✗ | 支持但退化 | **✓ stable** |
| 增量 delete | 不支持 | ✗ | 支持但退化 | **✓ stable via batch consolidation** |
| 50 cycles 5% change recall | 95% → 90% | 95% → 88% | 95% → 89% | **95% 稳定** |
| Build speed | reference | reference | reference | **1.5-1.8× faster** |
| Search performance（static index）| state-of-art | state-of-art | reference | **同 Vamana α=1.2**（无损） |
| 工业实证 | hnswlib / Faiss / Milvus | Taobao | DiskANN | [FreshDiskANN](../systems/freshdiskann.md) |

## 与 wiki 已 ingest concept 的关系

### 与 [Vamana](./vamana.md)

FreshVamana 是 Vamana 的"streaming 升级"——同 graph 算法、同 RobustPrune、同 α-RNG 性质；新增 (1) 形式化 insert/delete 算法，(2) lazy delete + batch consolidation，(3) 实证 α > 1 在 streaming 场景的 navigability 保证。

### 与 [HNSW](./hnsw.md) / [NSG](./nsg.md) 的 delete 失败原因

HNSW M=16 / NSG R=8 都用 α=1 implicit RobustPrune（aggressive pruning）→ graph 极稀疏 → 删点失去 navigability。这是 [hnsw.md] / [nsg.md] 已 flag 的 "不支持 delete" Open Q 的根因——FreshVamana 提供唯一 graph 路径解。

### 与 [LIRE](./lire.md)（cluster-path 姐妹工作）

[LIRE](./lire.md) 在 [SPFresh](../systems/spfresh.md) 中实现 SPANN cluster 的 in-place 增量 rebalance；FreshVamana 在 [FreshDiskANN](../systems/freshdiskann.md) 中实现 Vamana graph 的 streaming update。两者 dual：

| | LIRE (cluster path) | **FreshVamana (graph path)** |
|---|---|---|
| 论文 | SPFresh SOSP 2023 | FreshDiskANN arXiv 2021 |
| 团队 | Microsoft Research（Yuming Xu et al.） | Microsoft Research India（Singh et al.）|
| 索引 base | SPANN | Vamana |
| 数据组织 | inverted lists（cluster） | proximity graph |
| 增量 update 机制 | split/merge/reassign cluster | RobustPrune α>1 + lazy delete consolidation |
| 必要条件 | 2 NPA conditions | α-RNG property |
| 实测 stability | 100 days × 1% daily | 50 cycles × 5%/10%/50% |
| 1B SIFT 实证 | ✓ | ✓ |

→ FreshDiskANN 早 2 年（2021 vs 2023）；SPFresh 等价对位 cluster path——这是 wiki 内首次明示这种 dual 关系。

### 与 [FilteredVamana](./filtered-vamana.md) / [Block Shuffling](./block-shuffling.md)

Vamana 的三个独立后续延伸：
- **FreshVamana** [singh-2021]：streaming update（**本 page**）
- **FilteredVamana** [gollapudi-2023]：filter-aware build
- **Block Shuffling** [wang-2024-starling]：disk layout 重排

→ 三者**正交**，理论上可叠加：FreshVamana + FilteredVamana + Block Shuffling 在同一 SSD-resident streaming filtered index——wiki 内 zero coverage 实证。

## Open Questions

- **HNSW / NSG 的 α-RNG 等效改造**：FreshVamana 因 Vamana 已有 α 参数，自然引入 α=1.2；HNSW / NSG 没有等价参数——能否给 HNSW 加类似机制？理论上 yes（任何 RobustPrune-style edge selection 都可加 α），但工业实现未做
- **α-RNG 在 ACORN-γ predicate-agnostic graph 上的行为**：[ACORN](./acorn.md) γ-density factor 与 α-RNG 是否冲突？wiki 未覆盖
- **多 graph 算法（NSG / HNSW / Vamana）在 fresh-ANNS 下的 head-to-head**：FreshDiskANN paper 仅证明 α=1 失败 + α>1 work；不同 α-augmented graph 算法的对比未做
- **Concurrent insert/delete 的正确性证明**：fine-grained locking OK 但是否完全 serializable？论文实证 OK 但形式化证明 lacking
- **α 选择的理论指导**：α=1.2 是经验调参；论文 §4 仅给 "α > 1 必要"，未给最优 α 的 closed form
- **Graph 稀疏化的极限**：α 足够大时 graph 变 dense → search performance 下降；trade-off 边界未深入
- **跨 dataset 的 α 稳定性**：4 个 dataset 都用 α=1.2 work；高度聚类 / 极端分布 dataset 的 α 选择未实证
- **FreshVamana + [Block Shuffling](./block-shuffling.md)**：streaming update 后 OR(G) 漂移；定期重新 block shuffle 的开销 vs 收益未量化
- **embedding model 升级**：与 wiki 全 frontier 一致——quantization layer 同样需重 build；FreshVamana 不解决跨 model

Cited by: 待 query 引用
