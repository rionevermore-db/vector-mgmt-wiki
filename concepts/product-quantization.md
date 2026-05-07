---
title: Product Quantization（PQ / IVFADC）
type: concept
sources: [jegou-2011-pq, guo-2019-scann, johnson-2017-faiss-gpu, douze-2024-faiss-library, subramanya-2019-diskann]
related: [hnsw.md, proximity-graph.md, scann.md, warpselect.md, vamana.md, ../systems/faiss.md, ../systems/diskann.md, ../topics/mips-vs-l2-nn.md, ../topics/gpu-vs-cpu-ann.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/pq-sift-recall.md, ../benchmarks/hnsw-vs-faiss-200m-sift.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md, ../benchmarks/diskann-sift1b.md]
created: 2026-05-07
updated: 2026-05-07
---

# Product Quantization

**TL;DR**: 把 D 维向量切成 m 段独立量化，得到 m·log₂(k*) 比特的短 code（典型 64 bit）；ADC 让未量化的 query 直接对 code 估距离；IVFADC 再加一个 coarse quantizer 形成倒排表，把扫描代价从 O(N) 降到 O(N·w/k')。Faiss IVFPQ 的理论原型。[jegou-2011-pq §III–IV]

## 提出背景

Hervé Jégou、Matthijs Douze、Cordelia Schmid（INRIA），IEEE TPAMI 2011。
解决的问题：十亿级图像检索（SIFT / GIST 描述符）下，LSH / FLANN 类算法依赖原始向量驻留内存做 re-ranking，**内存而非 CPU 是瓶颈**。[jegou-2011-pq §I]

- 64 bit code 一个向量 → 1B 向量 ≈ 8 GB，单机内存可装。
- 对比 Hamming embedding / spectral hashing：PQ 提供更多可区分距离值（不止"位差异"），并且天然估计期望平方距离。[jegou-2011-pq §I 末尾]

## 三层结构

### Layer 1 — Product quantizer（编码）

- 输入向量 x ∈ ℝ^D 切成 m 段，每段 D*=D/m 维。
- 每段独立做 k-means 得到 k* 个中心；x 编为 m 个 index 拼接的 code，长度 l = m·log₂(k*)。
- 等效中心数 (k*)^m，存储只需 m·k*·D* 个 float。例：m=8, k*=256, D=128（SIFT）→ 64 bit code，2^64 个等效中心，codebook 仅 256·128 = 32K float。[jegou-2011-pq §II.B + Table I]
- **维度分组很重要**：把相关分量分到同一个子量化器。SIFT m=4 时 natural order recall@100=0.593，random=0.501，按 patch 结构化分组=0.640。[jegou-2011-pq §V.C, Table IV]

### Layer 2 — Distance computation（SDC vs ADC）

| | SDC（symmetric） | ADC（asymmetric） |
|---|---|---|
| query 是否量化 | 是 | **否** |
| 距离来源 | 预计算 (k*)² 表查表 | 在线对每段算 query→k* 中心距离 |
| 距离误差 | MSDE ≤ 2·MSE | MSDE ≤ MSE |
| query 端开销 | 仅 code | k*·D 浮点距离 |
| 推荐 | 否 | **是**（同等代价、更准） |

[jegou-2011-pq §III.A–B + Table II]

ADC 的核心：query 不丢精度，只有 database 端被量化。这是 PQ 比 hashing 类算法天然更准的关键。

### Layer 3 — IVFADC（倒排表 + ADC）

- 加 coarse quantizer q_c（独立 k-means，k' ∈ [1k, 1M]）将空间切成倒排表，库向量 y 存为 `(q_c(y), q_p(r(y)))`，其中残差 `r(y) = y − q_c(y)`。[jegou-2011-pq §IV]
- 量化残差而非原向量：残差能量小，PQ 误差更低。
- 查询时 query x 分配到 w 个最近 coarse cell（multiple assignment），只扫这些 cell 内的 code。扫描比例 ≈ w/k'。
- 每条记录 = 8–32 bit ID + m·log₂(k*) bit code。[jegou-2011-pq §IV.B]

## 关键性质 / 复杂度

| 阶段 | 时间 | 内存 |
|---|---|---|
| 编码（每库向量） | O(k*·D) 浮点 | l 比特 / 向量 |
| ADC 查询 | O(k*·D) 表预算 + O(N·m) 距离估计 | 全表扫 |
| IVFADC 查询 | O(k'·D) 粗量化 + O(N·m·w/k') 距离估计 | w/k' 比例扫 |

64 bit code + 8 bit ID = 9 字节 / 向量 → 2 B 向量约 18 GB，单机可装。[jegou-2011-pq §V.F]

## 关键参数

| 参数 | 典型值 | 作用 |
|---|---|---|
| `m` | 4–16 | 子量化器数；越大压缩比越高、误差越大 |
| `k*` | 256（=2^8） | 每子量化器中心数；几乎都用 256（一字节 index） |
| code length `l` | m·log₂(k*) | 总编码长度，常见 64 / 128 bit |
| `k'` | 1k–1M | 粗量化器中心数 |
| `w` | 1–64 | multiple assignment，召回-速度调节钮 |

[jegou-2011-pq §II.B, §IV, §V.B]

经典默认：`m=8, k*=256, k'=8192, w=8` —— 论文 SIFT 实验的代表配置。

## 与同类对比

| | PQ / IVFADC | [HNSW](./hnsw.md) | LSH (E2LSH) | FLANN (KD-tree / HKM) |
|---|---|---|---|---|
| 内存（1M SIFT） | <25 MB | 数百 MB | 需原向量驻留 | >250 MB |
| 召回-速度曲线（SIFT 1M） | 优 | 最优 | 中 | 中 |
| 适合规模 | 1B+（内存敏感） | 1M–100M（内存充足） | 中等 | 1M 级 |
| 支持删除 / 更新 | 是 | 否 | 是 | 部分 |
| 分布式 | 是（按 coarse cell 分片） | 困难 | 是 | 是 |

[jegou-2011-pq §V.D + Fig 10]；HNSW 一侧详见 [HNSW vs Faiss PQ on 200M SIFT](../benchmarks/hnsw-vs-faiss-200m-sift.md)。

## 典型实现

- 论文随附 C 实现：[INRIA texmex group](http://www.irisa.fr/texmex/people/jegou/ann.php)。
- **Faiss `IndexIVFPQ`**（Facebook Research）—— 工业事实标准，论文方案的直接工程化版本，加了 SIMD 距离表查询、GPU 实现、OPQ 预处理等。
- 后续重要变体（未在本论文）：OPQ（Optimized PQ）、LOPQ、IMI（Inverted Multi-Index）、PQFastScan、[ScaNN（Anisotropic VQ）](./scann.md)。

## 后续演化：Score-aware loss（[ScaNN](./scann.md)）

PQ 论文优化 reconstruction error `||x − x̃||²`，隐含假设所有 (q, x) 对等权重。2019 年 Guo et al.（Google）指出对 MIPS 任务这是次优的：

- 高 `<q,x>` 对更可能成为 top-k，对它们的量化误差应被加重；
- 残差中"平行于 x 的分量"对 MIPS 排序影响更大（相比正交分量）。

[ScaNN](./scann.md) 把 PQ 的 loss 改为：
`ℓ = h_∥·||r_∥||² + h_⊥·||r_⊥||²`，其中 h_∥ ≥ h_⊥（[guo-2019-scann Theorem 3.3]）。

工程改造成本极低：assignment 与 update 步骤替换为 anisotropic 版本，闭式解 [guo-2019-scann Theorem 4.2]；当 h_∥ = h_⊥ 时退化回 k-means update（即原始 PQ）。

效果：Glove1.2M 上 200 bit code Recall1@10 从 0.83 提升到 0.91。[guo-2019-scann Fig 3a] 详见 [ScaNN](./scann.md) 与 [topics/mips-vs-l2-nn.md](../topics/mips-vs-l2-nn.md)。

## GPU 实现要点（Faiss-GPU IVFADC）

[johnson-2017-faiss-gpu] 给出 IVFADC 在 GPU 上的具体 layout，把 SIFT1B 的 ANN 时间压到 17.7 μs/query（单 Titan X，比前作 8.5× 快）。关键技术：

- **PQ lookup 表放 shared memory**：每 query 一次性预算 b 个 256-元素表 T₁..T_b（每个 ~1KB）；扫倒排表时用 lookup-add 而非 multiply-add（bandwidth-bound）。[johnson-2017-faiss-gpu §5.2]
- **距离计算 + k-selection 融合 kernel**：扫倒排表的同一 kernel 直接调 [WarpSelect](./warpselect.md)，避免中间矩阵 D' 写回，节省 25% 时间。[johnson-2017-faiss-gpu §5.1, §5.3]
- **三项分解**：`||x − q(y)||² = ||q₂(...)||² + 2<q₁, q₂(...)> + ||x − q₁||² − 2<x, q₂(...)>`，前两项 query-independent 可预算，第三项 q₁-only，只有第四项需逐 list 算。[johnson-2017-faiss-gpu Eq 11]
- **multi-GPU**：
  - **Replication**（每 GPU 一份完整 index）→ 吞吐近线性扩展；
  - **Sharding**（index 切到多 GPU）→ 内存换吞吐，但末端 k-select 合并降低效率；
  - 二者可组合（S × R GPUs）。[johnson-2017-faiss-gpu §5.4]

详见 [Faiss-GPU on SIFT1B / DEEP1B / YFCC100M](../benchmarks/faiss-gpu-sift1b-deep1b.md) 与 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md)。

## Quantizer 家族中的位置（[Faiss 综述 §4](../systems/faiss.md)）

[douze-2024-faiss-library §4] 把所有量化方法拉到一个 hierarchy：

```
binary (1-bit scalar)
  ⊂ scalar quantizer (per-dim, 4/6/8 bit)
    ⊂ product quantizer (PQ — 本 page)
      ⊂ product-additive quantizer (PRQ, PLSQ)
        ⊂ additive quantizer (RQ, LSQ)
          ⊂ general MCQ
```

每层比上一层有更多自由度（更准）但也更贵（更慢、训练更多）。**PQ 在这个层级里相当于"M 个 1-level additive quantizer"的特例**。

代表算法：

- **Residual Quantizer (RQ)**：依次量化残差 [Chen 2010]
- **Local Search Quantizer (LSQ)**：simulated annealing 优化 codebook [Martinez 2016, 2018]
- **Product Residual Quantizer (PRQ)**：M 子向量 × 各自 RQ
- **Product LSQ (PLSQ)**：M 子向量 × 各自 LSQ
- 全部支持类似 ADC 的距离查询（[douze-2024-faiss-library Eq 14]）

经验（[douze-2024-faiss-library §4.4 + Fig 3]）：

- 小 code（<32 byte）下 LSQ / RQ 优于 PQ；
- 大 code（>64 byte）下 PRQ / PLSQ 接管；
- [ScaNN](./scann.md) 的 anisotropic loss 与所有这些方法**正交**，可以叠加。

## DRAM-PQ + SSD-FullPrecision 混合模式（[DiskANN](../systems/diskann.md)）

[subramanya-2019-diskann §3] 提出新的 PQ 使用模式：

- **PQ codes 仍放内存**做距离估计与图遍历导航
- **全精度向量放 SSD**，与图节点同扇区，每次邻居读"顺手"取回（[subramanya-2019-diskann §3.5]）
- 最终 ranking 用全精度向量，**不用 PQ ADC 距离**

效果：把 PQ 的 recall 上限（量化失真天花板，~62% on SIFT1B + IVFOADC+G+P-32）打到 100%（DiskANN single 在 SIFT1B 上 98.68% [subramanya-2019-diskann §4.4]）。代价：每查询 SSD 访问数 = hop 数 ~10 个，总延迟 ~ms 级。

这与 PQ 论文 [jegou-2011-pq §III] 设想的"PQ ADC 是最终距离"不同 —— 在 [DiskANN](../systems/diskann.md) 里 PQ **只做导航不做 ranking**。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

## Open Questions

- 如何让 PQ 量化器学到与查询分布对齐而非仅与 database 分布对齐？论文的 ADC 仍假设 query 与 database 同分布。**部分回答**：[ScaNN](./scann.md) 的 score-aware loss 显式建模 query 分布并按 inner product 加权 [guo-2019-scann §3] —— 但仅针对 MIPS，L2-NN 通用版仍开放。
- 维度分组的自动化：论文提到 minimum sum-squared residue co-clustering [30 in jegou-2011-pq] 是潜在方向，但未实施。[jegou-2011-pq §V.C 末尾]
- IVFADC 的 coarse quantizer 用更优结构（如 IMI、HNSW-as-coarse-quantizer）能否进一步降低 k'·D 的查询开销？论文 §V.E 末尾承认对大 k' 用 hierarchical quantizer，工业界已有 HNSW + PQ 混合方案。**部分工程化**：[Faiss-GPU](./warpselect.md) [johnson-2017-faiss-gpu] 把 IVFADC 整体迁移到 GPU 后，coarse quantizer 反而变成相对小的开销（GPU brute-force 算 k'×D 极快），实际工程更关注 fused kernel 与 PQ lookup 表布局。
