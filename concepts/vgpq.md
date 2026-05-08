---
title: VGPQ（Voronoi Graph Product Quantization）
type: concept
sources: [wei-2020-analyticdb-v]
related: [./product-quantization.md, ./scann.md, ../systems/analyticdb-v.md, ../topics/disk-vs-memory-ann.md, ../topics/attribute-filtering.md]
created: 2026-05-08
updated: 2026-05-08
---

# VGPQ

**TL;DR**: ADBV 自研的 IVFPQ successor。在 IVFPQ 的 Voronoi cell 上**进一步用相邻 centroid 的 midpoints 切成 subcells** —— query 时只扫覆盖 query 邻域的少量 subcells（而非整个 cell）。比 IVFPQ same recall 下**response time 显著更低**；construction time -10%（同 n_clusters）；index size 同 IVFPQ。在 SIFT1B / Deep1B / **AliCommodity 830M × 512-d** 三个 dataset 上系统优于 IVFPQ。是 [AnalyticDB-V](../systems/analyticdb-v.md) batching layer 的核心算法。[wei-2020-analyticdb-v §4.2]

## 提出背景

[wei-2020-analyticdb-v §4.1, Fig 6]

[IVFPQ](./product-quantization.md) 的痛点：
- IVFPQ 用 k-means 把空间切成 n_clusters Voronoi cell
- query 时找最近的 nprobe 个 cell + 扫这些 cell 内所有 PQ codes
- 但**很多 cell 内 PQ codes 远离 query**——查询浪费

ADBV 团队观察：**Voronoi cell 内距 query 较远的部分是确定性可剪枝的**——基于"垂直平分超平面" geometric argument。

具体例子（论文 Fig 6）：
- Query q 落在 cell C_0
- IVFPQ 查 q 的 top-3：扫 C_0 + C_2（避免漏 C_2 内候选）所有 PQ codes（论文图 6a）
- VGPQ 观察：C_0 沿 C_0–C_1, C_0–C_2, ..., C_0–C_7 的 7 条边各有 midpoint；用 midpoints 把 C_0 切成 7 个 subcells；每 subcell 标记"距 q 的最大距离"
- 仅扫**覆盖 q 邻域的 6 个 subcells**（图 6b 阴影）——比 IVFPQ 整个 cell 少很多

## 三层结构

### Layer 1：Voronoi diagram on IVFPQ centroids（§4.2）

```
1. 先做 IVFPQ：n_clusters centroids C_0, ..., C_{n-1}
2. 对每 centroid C_i，找其 n_subcells 个最近 neighbor centroids
3. C_i 与每个 neighbor C_j 形成边；取 midpoint M_{i,j} = (C_i + C_j) / 2
4. 每条边用 M_{i,j} 把 C_i 的 Voronoi cell 切成两半
5. 多条边交叉切割 → 把 C_i 的 Voronoi cell 分成 b 个 subcells
6. 每 subcell B_{i,j} 关联 anchor centroid C_i 与 neighbor centroid C_j
```

### Layer 2：Subcell 距离计算（§4.2）

定义 subcell B_{i,j} 与 query q 的距离 d(q, B_{i,j}) = q 到线段 (C_i, C_j) 中点的距离。论文论证：

```
对 subcell B_{i,j}：
   subcell 内任意点 p 满足：
      D(p, C_i) < D(p, M_{i,j})  （即 p 比 M 更近 C_i——subcell 定义）
   query 仅访问满足"d(q, B_{i,j}) ≤ d(q, B 第 b 名)"的 subcells
```

**实际效果**：原来扫 1 整个 cell 现在仅扫 6 个 subcells（论文 Fig 6b 例）—— **constant factor 提升**。

### Layer 3：Subcell 内 PQ scan（§4.2）

每 subcell 内：
- PQ codes 与 IVFPQ 同样（每向量一个 m-byte code）
- ADC 距离估计同 IVFPQ
- 但只在覆盖 query 邻域的 subcells 内扫

**关键**：VGPQ index size 与 IVFPQ 同（PQ codes 没增加），仅 metadata（subcell 关联表 + Voronoi 拓扑）多一些。

## 关键参数

[wei-2020-analyticdb-v §4.2 Algorithm 1]

| 参数 | 典型值 | 作用 |
|---|---|---|
| `n_clusters` | 4096 / 8192 | IVFPQ centroid 数；同 IVFPQ |
| `n_subcells` | 32 / 64 / 128 | 每 cell 切成多少 subcells（取邻居 centroid 数） |
| `b` | 同 n_subcells | 每 cell 内 subcell 数 |
| `s` | query 时调 | 访问的 anchor centroids 数 |
| `b'` | query 时调 | 每 anchor 内访问的 subcells 数 |

[wei-2020-analyticdb-v §6.2 Table 2]：n_clusters=4096 + n_subcells=64 是经验默认。

## 与 [PQ / IVFPQ](./product-quantization.md) 的对比

| | IVFPQ | **VGPQ** |
|---|---|---|
| Coarse partition | k-means Voronoi cell | k-means Voronoi cell |
| Cell 内细分 | ✗（整个 cell scan） | **✓ midpoints 切成 subcells** |
| Query 扫描范围 | nprobe 个完整 cell | **s 个 anchor cell × b' 个 subcells** |
| Index size | 同基线 | **同 IVFPQ** |
| Construction time | baseline | **-10%**（同 n_clusters） |
| Recall vs response time | baseline | **显著更优**（SIFT1B / Deep1B / AliCommodity 实测） |

## 与 [ScaNN](./scann.md) 的关系

| | ScaNN | VGPQ |
|---|---|---|
| 创新维度 | quantization loss（score-aware） | partition geometry（Voronoi + subcells） |
| 主攻任务 | MIPS | L2 + 内积 + Hamming（ADBV 全支持） |
| 与 IVFPQ 关系 | 替换 quantization loss | 在 Voronoi 上加几何剪枝 |
| 集成系统 | Google ScaNN / Faiss FastScan | AnalyticDB-V batching layer |
| 复制思路 | anisotropic loss | midpoint geometric pruning |

→ ScaNN 与 VGPQ 是**正交的 IVFPQ 改进方向**——分别从 quantization loss 与 cell partitioning 入手。理论上可叠加，但论文未尝试。

## 实验数据

[wei-2020-analyticdb-v §6.2 Table 2 + Fig 10]

### Construction time + index size（AliCommodity）

| Method | Time (min) | Size (GB) |
|---|---|---|
| IVFPQ(4096) | 155 | 112 |
| IVFPQ(8192) | 199 | 112 |
| **VGPQ(4096, 64)** | **144** | 112 |
| VGPQ(8192, 64) | 178 | 112 |
| VGPQ(8192, 128) | 182 | 112 |

→ 同 n_clusters，VGPQ build time **-10%**。Index size 完全相同。

### Recall vs Response Time（SIFT1B / Deep1B / AliCommodity）

[wei-2020-analyticdb-v Fig 10]：
- 在三个 dataset 上 VGPQ 曲线**全程在 IVFPQ 之上**（同 response time recall 更高）
- AliCommodity 上 VGPQ(4096, 32/48/64) 曲线明显优于 IVFPQ(1024/2048)
- 优势随 n_subcells 增加（直到一定阈值后 marginal）

### Clustering-based partitioning + VGPQ

[wei-2020-analyticdb-v Fig 11]：
- SIFT1B_512p（512 partition）：从 512 → 3 partition pruning，仍 95% recall on top-50
- 总 QPS **100×+ 提升**
- Deep1B_512p 类似 10× 提升 ideally

## 在 [AnalyticDB-V](../systems/analyticdb-v.md) 中的角色

[per systems/analyticdb-v.md "Lambda 三层框架"]

VGPQ 仅在 batching layer 使用：
- Streaming layer 用 HNSW（real-time，新数据小）
- **Batching layer 用 VGPQ**（baseline 大数据，offline build）
- Serving layer merge 两层结果

ADBV 的 4-plan CBO 中：
- Plan B 用 PQ + ADC
- **Plan C 用 VGPQ Knn Bitmap Scan**
- **Plan D 用 VGPQ Knn Scan + filter**
- → VGPQ 是 ADBV hybrid query optimization 的"中坚算法"

## Open Questions

- **VGPQ 在其他系统的 portability**：算法本身公开，但 wiki 内仅 ADBV 集成；为什么 Faiss / Milvus / Pinecone 不直接采用？可能 Voronoi 几何在高维下退化（论文承认低维优势更大；高维 cell 形状非凸超复杂，subcell 切割效果递减）
- **n_clusters / n_subcells 的最优组合**：论文 default 4096/64 经验；不同数据分布最优值未量化
- **Voronoi diagram 在高维下的数值稳定性**：高维下 Voronoi cell 形状极不规则，midpoint 切割可能偏离最优——论文未深入
- **VGPQ + ScaNN anisotropic loss 叠加**：理论可行未试
- **Update under VGPQ**：增量插入需重建 Voronoi（centroid 变 → midpoints 变 → subcell 边界变）；ADBV 用 lambda 框架避开（streaming HNSW 处理增量），但纯 VGPQ in-place update 未解
- **VGPQ vs HNSW 实测比较**：论文 §6 主要 vs IVFPQ；VGPQ vs HNSW 的 recall / latency / build / memory 全 trade-off 未独立比较

Cited by: 待 query 引用
