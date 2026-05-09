---
title: Vamana（α-controlled graph）
type: concept
sources: [subramanya-2019-diskann, wang-2024-starling, singh-2021-freshdiskann, ootomo-2023-cagra]
related: [hnsw.md, nsg.md, proximity-graph.md, product-quantization.md, filtered-vamana.md, block-shuffling.md, freshvamana.md, cagra-graph.md, ../systems/diskann.md, ../systems/starling.md, ../systems/freshdiskann.md, ../systems/cagra.md, ../topics/disk-vs-memory-ann.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/gpu-vs-cpu-ann.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md, ../benchmarks/freshdiskann-streaming-sift800m.md, ../benchmarks/cagra-vs-hnsw-ggnn-ganns.md]
created: 2026-05-07
updated: 2026-05-09 (CAGRA)
---

# Vamana

**TL;DR**: 单层有向 proximity graph，核心是 RobustPrune 邻居选择中引入 α≥1 参数 + 两遍构建。α>1 让图更稀疏但保留长程边，使 graph diameter 显著小于 [HNSW](./hnsw.md) / [NSG](./nsg.md)。在 SIFT1M / GIST1M / DEEP1M 上击败 HNSW 与 NSG，是 [DiskANN 系统](../systems/diskann.md) 的内存层算法基础。[subramanya-2019-diskann §2 + Fig 3]

## 提出背景

Suhas Subramanya, Devvrit, Rohan Kadekodi, Harsha Vardhan Simhadri, Ravishankar Krishnaswamy（Microsoft Research India + UT Austin + CMU），NeurIPS 2019。论文早期投稿名 "Rand-NSG"，发表时改为 DiskANN（NeurIPS proceedings URL 还保留旧名）。

针对的问题：现有 graph ANN 算法（[HNSW](./hnsw.md)、[NSG](./nsg.md)、FANNG）都隐式使用 α=1（RNG 风格的边选择），缺少调节图直径的旋钮。这在 SSD 部署场景下成为瓶颈 —— 每跳 = 1 次磁盘读，大直径意味着多次 round-trip。

## RobustPrune（核心创新）

[subramanya-2019-diskann Algorithm 2]

```
RobustPrune(p, V, α, R):
  V ← V ∪ N_out(p) \ {p}
  N_out(p) ← ∅
  while V ≠ ∅:
    p* ← argmin_{p'∈V} d(p, p')   # 当前最近候选
    N_out(p) ← N_out(p) ∪ {p*}
    if |N_out(p)| = R: break
    for p' ∈ V:
      if α · d(p*, p') ≤ d(p, p'):
        remove p' from V
```

**关键差异**：与 RNG / [HNSW Alg 4](./hnsw.md) 相比，多了 α 系数。

- α = 1：等价于 SNG（Sparse Neighborhood Graph，Arya & Mount 1993）/ RNG 的边选择 —— 删除"被 p* 主导的"候选 p'。
- α > 1：放宽删除条件，**保留更多远邻**，结果是图更稀疏 + 含长程边。

直觉：α 控制"在 p 邻居 p* 周围多大半径内的其他候选会被剪掉"。α=1 时半径恰好 d(p,p*)（RNG 定义）；α=2 时半径减半，留下的远邻更多。

## Vamana 索引算法（两遍构建）

[subramanya-2019-diskann Algorithm 3]

1. **初始化** G 为随机 R-regular 有向图（与 HNSW 空图、NSG 近似 kNN graph 都不同）
2. **找 medoid** s（数据集中位向量），作为搜索固定起点
3. **第一遍**（α = 1）：对随机排列的每个 p
   - GreedySearch(s, x_p, 1, L) → V = 路径上所有访问点
   - RobustPrune(p, V, α=1, R) → 设 p 的新邻居
   - 加反向边：对每个 p' ∈ N_out(p) 加边 (p', p)；若 p' 出度超 R 则对 p' 也跑 RobustPrune
4. **第二遍**（α = 用户设置，通常 1.2–2）：重复 step 3 但用更大 α

**为什么两遍**：第一遍建出一个 RNG 近似图；第二遍在它基础上引入长程边，得到更小直径。论文实证两遍比一遍 + 大 α 更准更快。

## 关键参数

| 参数 | 典型值 | 作用 |
|---|---|---|
| `α` | 1.0 → 2.0 | 第二遍的边密度 / diameter 调节钮 |
| `L` | 100–500 | GreedySearch 候选列表大小（构建时） |
| `R` | 64–128 | 每点最大出度 |
| 起点 s | medoid | 固定 entry point（与 NSG 类似） |

[subramanya-2019-diskann §4.1]

## 与 HNSW / NSG 的差异

| | Vamana | [HNSW](./hnsw.md) | [NSG](./nsg.md) |
|---|---|---|---|
| 层数 | 单层 | 多层 | 单层 |
| α 参数 | **可调（核心创新）** | 隐式 α=1 | 隐式 α=1 |
| 长程边来源 | α>1 + 平面图 | 顶层 hierarchy | medoid + RNG cut |
| 初始图 | 随机 | 空 | 近似 kNN graph |
| 构建遍数 | **2** | 1 | 1 |
| Diameter 随 max degree | **显著降** [Fig 2c] | 平稳 | 平稳 |
| DEEP1M 构建时间 | 149 s | 219 s | 480 s（含 kNN graph） |

[subramanya-2019-diskann §2.4 + Fig 2c + Fig 3]

## 性能对比（million-scale，[Fig 3]）

在 SIFT1M / GIST1M / DEEP1M 三个数据集上，**Vamana 的 100-recall@100 vs latency 曲线在 HNSW 与 NSG 之上**（recall 越高，差距越明显）。

> 这是首次有 graph 算法在 NSG 之后再次系统性击败 HNSW —— [fu-2017-nsg §4.1] 之后的下一个 SoTA 推进。

详见 [DiskANN on SIFT1B benchmark](../benchmarks/diskann-sift1b.md)。

## 工业意义

Vamana 本身是 in-memory 算法，但**它的小 diameter 是 [DiskANN 系统](../systems/diskann.md) 在 SSD 上跑 5000 QPS @ <3ms 的关键**：

- SSD 单次随机读 ~几百 μs；目标 <5ms 总延迟 → 最多 ~10 跳
- Vamana 经验 hop 数比 HNSW / NSG 少 2-3× → 直接换算成更低 SSD 访问数

详见 [DiskANN 系统设计](../systems/diskann.md) 与 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

## 典型实现

- 作者实现：[`microsoft/DiskANN`](https://github.com/microsoft/DiskANN)（C++，含 Vamana in-memory + DiskANN SSD 双模式）
- [Faiss](../systems/faiss.md) 通过 `IndexNSG` 提供 NSG 但**不直接支持 Vamana**；DiskANN 是独立生态

## 后续演化：Streaming insert/delete（[FreshVamana](./freshvamana.md) + [FreshDiskANN](../systems/freshdiskann.md)）

[singh-2021-freshdiskann §4] Microsoft Research India + CMU（Subramanya 等 DiskANN 作者）2021 论文提出 **FreshVamana**——把 Vamana 从 static index 升级为支持 streaming insert + delete + recall 不退化的算法。**核心 insight**：Vamana 的 **α 参数（α > 1 RobustPrune）从"性能调节钮"升级为 fresh-ANNS 必要条件**——α=1（HNSW/NSG implicit default）下 graph 过稀疏，删点失去 navigability，recall 持续下降；α=1.2 下 50 cycles × 5%/10%/50% change 后 recall 95%+ 稳定。

**FreshVamana 是 strict superset of static Vamana**：
- 支持 streaming（static Vamana 不支持）
- Build 时间 1.48-1.83× faster（SIFT1M 32.3s → 21.8s）
- Recall 同 static Vamana α=1.2

→ 逻辑上 FreshVamana 应替代 default Vamana 算法。详见 [concepts/freshvamana.md](./freshvamana.md) 与 [systems/freshdiskann.md](../systems/freshdiskann.md)。

> **wiki 解读**：FreshVamana 是 Vamana 在 streaming 场景的最大延伸——比 [Filtered-DiskANN](./filtered-vamana.md) 的 attribute filtering 延伸**更基础**（algorithm 层 + system 层都改），且证明 α 参数比之前认知的更深刻。

## 后续演化：Disk layout 优化（[Starling](../systems/starling.md) + [Block Shuffling](./block-shuffling.md)）

[wang-2024-starling §4-§6.7] Zilliz/Milvus 团队 SIGMOD 2024 论文提出 **Starling-Vamana**——在 Vamana 构造的 disk graph 之上做 [block shuffling](./block-shuffling.md)（NP-hard 问题，BNF 启发式 default）+ in-memory navigation graph + block search。**算法本身不变**，只重排 vertex 到 disk block + 减少搜索路径长度。Vamana α-controlled RobustPrune 的 graph topology 保持完整。

效果（BIGANN 33M segment 配置）：Starling-Vamana 比 Disk-Vamana ANNS 2× 快、RS 43.9× 快；vertex utilization ratio 从 6.25% → 34%（5×）。详见 [systems/starling.md](../systems/starling.md) 与 [benchmarks/starling-vs-diskann-spann-on-segment.md](../benchmarks/starling-vs-diskann-spann-on-segment.md)。

> **wiki 解读**：Starling 是 Vamana 在 segment-level disk DBMS 场景的最大延伸——比 [Filtered-DiskANN](./filtered-vamana.md) 的 attribute filtering 延伸**更基础**（layout 层而非 graph 算法层），且与 FilteredVamana 正交可叠加（理论上 FilteredVamana + Starling block shuffling 同时使用未实证）。

## 后续演化：Filter-aware（[FilteredVamana](./filtered-vamana.md)）

[gollapudi-2023-filtered-diskann §3] 把 Vamana 的 RobustPrune α 系数加 **label intersection 检查**：FilteredVamana / StitchedVamana 两算法**首次把 label 信息 baked-in 到 graph 构造本身**——RobustPrune 时检查 `F_p* ⊃ F_p'` 来决定是否 prune。Microsoft sponsored ads A/B test +34.61% clicks / +48.95% revenue（[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]）。这是 [Vamana](#) 在 attribute filtering 维度的最大延伸。详见 [filtered-vamana.md](./filtered-vamana.md)。

## Open Questions

- **α 的最优值是经验调参**：论文给推荐区间 1.2-2 但没给理论指导；不同数据分布下的最优 α 是开放问题。
- **两遍构建的必要性**：第一遍 α=1 + 第二遍 α>1 是经验式发现；理论上单遍是否能匹配？论文未深入。
- **medoid entry point 的鲁棒性**：与 NSG 一样，单一 entry point 假设数据集有 well-defined 中心；高度聚类数据下 medoid 可能落到边缘。
- ~~**不支持增量**：与 NSG 同样问题；动态数据需要重建。FreshDiskANN 是后继工作，wiki 尚未 ingest~~ **2026-05-09 ingest [singh-2021-freshdiskann] 已解**：[FreshVamana](./freshvamana.md) (α=1.2) 把 Vamana 升级为 streaming-ready；[FreshDiskANN](../systems/freshdiskann.md) 是其 disk-resident system 实现
- **Vamana + [Block Shuffling](./block-shuffling.md) + 增量数据**：Starling §7 提"static disk index + 动态 in-memory + 周期 merge"模式（与 FreshDiskANN StreamingMerge 同源）；FreshVamana streaming insert 后 OR(G) 漂移；周期触发 block shuffling 重跑。摊销成本未量化。
- **Vamana 与 RaBitQ 集成**：[gao-2024-rabitq §4] 明示 graph-based 集成 future work；Vamana + RaBitQ 替代 PQ short codes for routing 是 logical 实验方向
- **Vamana 在 GPU 上的等效**：[CAGRA](./cagra-graph.md) [ootomo-2023] 是 NVIDIA-native GPU graph，与 Vamana 都用 fixed-style RNG pruning 但 CAGRA 用 **rank-based** 而非 distance-based（不需 distance 重算 → 1.9× faster + DEEP-100M 唯一可行）。Vamana α-RNG vs CAGRA rank-based reordering 是不同 first-principles design——CPU disk-resident vs GPU device memory 各自最优形态。详见 [systems/cagra.md](../systems/cagra.md) 与 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md)。
