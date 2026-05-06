---
title: Product Quantization（PQ / IVFADC）
type: concept
sources: [jegou-2011-pq]
related: [hnsw.md, proximity-graph.md, ../benchmarks/pq-sift-recall.md, ../benchmarks/hnsw-vs-faiss-200m-sift.md]
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
- 后续重要变体（未在本论文）：OPQ（Optimized PQ）、LOPQ、IMI（Inverted Multi-Index）、PQFastScan。

## Open Questions

- 如何让 PQ 量化器学到与查询分布对齐而非仅与 database 分布对齐？论文的 ADC 仍假设 query 与 database 同分布。
- 维度分组的自动化：论文提到 minimum sum-squared residue co-clustering [30 in jegou-2011-pq] 是潜在方向，但未实施。[jegou-2011-pq §V.C 末尾]
- IVFADC 的 coarse quantizer 用更优结构（如 IMI、HNSW-as-coarse-quantizer）能否进一步降低 k'·D 的查询开销？论文 §V.E 末尾承认对大 k' 用 hierarchical quantizer，工业界已有 HNSW + PQ 混合方案。
