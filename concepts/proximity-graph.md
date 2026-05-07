---
title: Proximity Graph（邻近图）
type: concept
sources: [malkov-2016-hnsw, fu-2017-nsg, johnson-2017-faiss-gpu]
related: [hnsw.md, nsw.md, product-quantization.md, nsg.md, ../topics/gpu-vs-cpu-ann.md]
created: 2026-04-30
updated: 2026-05-07
---

# Proximity Graph

**TL;DR**: 节点为数据点、边表达"邻近关系"的一类图。基于贪心遍历的 ANN 算法（[NSW](./nsw.md)、[HNSW](./hnsw.md)、FANNG、NNDescent 等）的共同基础数据结构。[malkov-2016-hnsw §2.1]

## 几个关键变体

| 变体 | 定义 | 性质 |
|---|---|---|
| **k-NN graph** | 每点连其 k 个最近邻 | 最简单；是 Delaunay 的近似 |
| **Delaunay graph** | a–b 之间存在边当且仅当存在某查询 q 使得"贪心从 a 走到 b"是最优路径 | 保证朴素贪心遍历总能到最近邻，但只能从空间结构构造、且高维下度数指数增长 [39 in malkov-2016-hnsw] |
| **Relative Neighborhood Graph (RNG)** | a–b 有边当且仅当不存在第三点 c 同时满足 d(c,a)<d(a,b) 且 d(c,b)<d(a,b) | Delaunay 的最小子图；HNSW Alg 4 的启发式选择得到其近似 [46 in malkov-2016-hnsw] |
| **Sparse neighborhood graph** | Arya & Mount 1993 [18 in malkov-2016-hnsw] | 与 RNG 思想类似，被 FANNG [47 in malkov-2016-hnsw] 使用 |
| **MSNET (Monotonic Search Network)** | 任意两点 p, q 之间存在至少一条单调路径（朴素贪心无回溯即可到达）[Theorem 1 in fu-2017-nsg] | Delaunay 是 MSNET；最小 MSNET 构造 O(n²log n + n²c)，不可大规模实施 [13 in fu-2017-nsg] |
| **MRNG (Monotonic RNG)** | RNG + 有向 + 按 index 排序的边选择，强制 NNG ⊂ MRNG [Definition 5 in fu-2017-nsg] | 最大出度独立于 n（Lemma 2）；期望搜索复杂度高维下 ≈ O(log N)；[NSG](./nsg.md) 是其工程化近似 |

## 为什么 ANN 用 proximity graph

1. **不需要预训练或量化**——只用元素之间的距离就能构造，与具体度量空间无关。
2. **贪心遍历就能近似 K-NN**——k-NN graph 上从任意点出发贪心走，结果是最近邻的好近似。[malkov-2016-hnsw §2.1]
3. **支持任意度量**（甚至非度量，如 JS 散度）——不依赖向量空间结构。HNSW 论文 §5.3 在多个非度量数据集上验证了这点。

## 与树 / 哈希 / 量化的对比

| 类别 | 代表 | 适合数据 | 核心限制 |
|---|---|---|---|
| **树** | KD-tree, VP-tree | 低维 (<20) | 维度灾难下退化为线扫 |
| **哈希** | LSH (FALCONN) | 高维欧氏 / 余弦 | 召回-速度 trade-off 较差 |
| **量化** | [PQ / IVFPQ](./product-quantization.md) (Faiss) | 十亿级、内存敏感 | 精度有损 |
| **Proximity graph** | NSW, HNSW, FANNG | 中高维、内存充足 | 内存开销大、删除困难 |

[malkov-2016-hnsw §2.1 + §5]

## 关键挑战

- **构造复杂度**——精确 Delaunay 不可达，k-NN graph 也是 O(N²) 朴素构造。NSW / HNSW 用增量插入 + 贪心搜索把构造摊到 O(N log N)。**GPU 路径反例**：[johnson-2017-faiss-gpu §6.5] 在 4 Titan X 上 35 分钟构造 95M 图的 k-NN graph（质量 0.8+）；NN-Descent 在 128-CPU 集群上 36.5M × 384-d 报告 108.7 小时 —— GPU brute-force + IVFADC 反而比图构造算法更快。详见 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md)。
- **聚类 / 簇间连通**——朴素 k-NN 在簇内打满，簇间几乎无边，搜索卡在簇边界。RNG 风格选择（HNSW Alg 4）显式保留跨簇边。[malkov-2016-hnsw Fig 2]
- **路径上节点度数随 N 增长**——纯 NSW 因此是 polylog 而非 log；HNSW 通过分层固定每层度数解决。

## Open Questions

- 高维下 RNG 的度数上界？HNSW 论文 §4.2 承认对 d≫1 没有证明，只有经验。
- 是否存在比 RNG 更稀疏、仍能保证贪心成功的 proximity graph？开放方向。
