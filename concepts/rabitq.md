---
title: RaBitQ（首个 unbiased + sharp error bound 的高维向量 quantization）
type: concept
sources: [gao-2024-rabitq]
related: [product-quantization.md, scann.md, vgpq.md, hnsw.md, vamana.md, relaxed-monotonicity.md, ../systems/faiss.md, ../systems/diskann.md, ../systems/spann.md, ../systems/milvus.md, ../systems/vbase.md, ../systems/analyticdb-v.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../topics/topk-vs-iterator-model.md, ../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]
created: 2026-05-08
updated: 2026-05-08
---

# RaBitQ

**TL;DR**: NTU Singapore 在 SIGMOD 2024 提出的 quantization 方法——**wiki 内首个同时拥有 unbiased estimator + sharp probabilistic error bound + 工业级 empirical accuracy** 的高维向量量化器 [gao-2024-rabitq §1, §3]。直接攻击 [PQ / OPQ / LSQ / ScaNN](./product-quantization.md) 等所有现有 quantizer 的"无 theoretical error bound"软肋——后者在某些真实数据集（MSong / Word2Vec）即使用 re-ranking 也 ≤60% recall，average relative error >50%。RaBitQ 的核心抽象是 **D-bit string + randomly transformed bi-valued hypercube codebook**——把每维量化为 ±1/√D 两个值，配合随机正交矩阵 P 把代码本均匀分布在单位超球面，距离 O(1/√D) **渐近最优**（[Alon-Klartag 2017]）。**关键工程后果**：error bound 让 IVF + RaBitQ 的 re-ranking **无需 K' 调参**——这是 quantizer 层对 [TopK 接口 K' 预测问题](../topics/topk-vs-iterator-model.md) 的"dual 攻击"，与 [VBASE](../systems/vbase.md) 的 query-engine 层 RM iterator 范式正交且可叠加。

## 提出背景

[gao-2024-rabitq §1]

PQ [jegou-2011-pq] 与变体（OPQ / LSQ / ScaNN / 加性量化）共同的根本弱点：

1. **Codebook construction 是 heuristic**——KMeans clustering on sub-segments，难以理论分析
2. **Distance estimation 是 biased**——直接把 quantized vector 当作 data vector 算距离，没有 error bound
3. **Re-ranking K' 调参 exhausting**——[gao-2024 §1] 直接引用 Faiss wiki "the tuning of the re-ranking parameter is often exhaustive and intertwined with many factors such as datasets and other parameters. Prior to the testing, there is no reliable way to predict the optimal setting"

[gao-2024-rabitq §1] 实测 PQ 在 MSong 上 ≥60% recall **even with re-ranking**——这意味着工业部署时无法预知什么数据集会"灾难性失败"。

LSH [21, 38, 39, 78-80] 提供 c-approximate 概率保证，但保证形式（"返回某个距离 ≤ (1+c)r 的向量"）对 ANN re-ranking **无用**——需要的是"per-data-vector 距离估计的误差界"。

## 核心算法

### 1. Codebook construction（§3.1）

**第一步：normalize 到单位超球面**

```
o := (o_r - c) / ||o_r - c||
q := (q_r - c) / ||q_r - c||
```

c 是 IVF cluster centroid。把无界 Euclidean 空间映射到有界单位球面——这是 theoretical guarantee 必需的（无界空间无法构造均匀分布的 codebook）。

**第二步：构造 deterministic codebook**

```
C := {+1/√D, -1/√D}^D    （即 hypercube vertices）
|C| = 2^D                  （超大！）
```

**关键性质**：所有 hypercube vertices 都是 unit vectors（因为 ||x||² = D · (1/√D)² = 1），**均匀分布在单位球面上**。

**第三步：随机化避免特定向量退化**

朴素的 deterministic C 对某些 query 退化（例如 (1, 0, ..., 0) 与所有 codebook vector 距离相同）。RaBitQ 应用随机正交矩阵 P：

```
C_rand := {Px : x ∈ C}
```

P 是 random orthogonal matrix（Johnson-Lindenstrauss Transform [49]）。Geometrically，C_rand 是 C 在 sphere 上的随机旋转——distribution-uniform。

### 2. Quantization code = D-bit string（§3.1.3）

对每个 normalized data vector o，找 C_rand 中最近的 vector 作 quantized vector。**关键观察**：内积 invariant under orthogonal transformation，所以可以**反向变换** o → P^{-1}o 然后在 C 中找最近：

```
x̄ := arg max_{x ∈ C} ⟨P^{-1}o, x⟩
```

由于 x 的每维只能是 ±1/√D，要最大化内积只需让 x 每维的 sign 匹配 P^{-1}o 对应维的 sign：

```
x̄_b[i] = 1 if (P^{-1}o)[i] >= 0 else 0    （D-bit string）
x̄[i] = (2 x̄_b[i] - 1) / √D                  （reconstruct ±1/√D）
ō := P x̄                                     （quantized vector in original space）
```

→ **每维仅 1 bit**，整个 vector 仅 **D bits**——比 PQ default 2D bits 紧 50%。

不显式存 C_rand（代码本只有 P 和 D-bit string；占 32D bits 远小于 raw vector 的 32D bits）。

### 3. Unbiased distance estimator（§3.2）

设需要估计 unit-vector 间内积 ⟨o, q⟩（即归一化后的余弦相似度）。**核心 insight**：研究 ō 与 o 的几何关系——⟨ō, o⟩ 在 D ∈ [10², 10⁶] 时严格收缩到 0.798-0.800（Gamma 函数闭式 [§3.2.1, footnote 5]）；⟨ō, e₁⟩（e₁ 是 q 在垂直 o 方向的分量）期望 0 + 高度集中（Lemma B.3）。

由此推导（[Theorem 3.2]）：

```
estimator: ⟨ō, q⟩ / ⟨ō, o⟩
E[⟨ō, q⟩ / ⟨ō, o⟩] = ⟨o, q⟩          （unbiased）
```

**Sharp probabilistic error bound** [Theorem 3.2 + Eq 14]：

```
P{ |⟨ō,q⟩/⟨ō,o⟩ - ⟨o,q⟩| > (√(1-⟨o,q⟩²) / ⟨ō,o⟩²) · (ε₀/√(D-1)) } ≤ 2 exp(-c₀ ε₀²)
```

→ failure probability **quadratic-exponential** in ε₀。**ε₀ = 1.9 across all 6 datasets**（论文 §5.2.4，无需调参）。

误差量级 **O(1/√D) w.h.p.**——根据 [Alon-Klartag 2017, ref 3] 这是 D-bit 短码的理论下界，RaBitQ **asymptotically optimal**。

### 4. Efficient implementation（§3.3）

**Single vector 距离估计**——对单个 data vector + 单个 query：

```
1. Pre-compute ⟨ō, o⟩ in index phase
2. Query phase: ⟨x̄, q'⟩ = ⟨P^{-1}q, x̄⟩  via bitwise AND + popcount
   （x̄ 是 D-bit binary, q' 是 B_q-bit unsigned int per dimension after randomized scalar quantization）
3. 距离估计 = (⟨x̄, q'⟩ scaled) → ⟨ō, q⟩ → original-space distance
```

[Theorem 3.3] B_q = Θ(log log D) 已足够；**B_q = 4 across all datasets** 实测。

**3× 快于 PQ LUT**（论文 §5.2.1）——因为 popcount 是 1 cycle SIMD instruction，PQ LUT 需要 RAM lookup。

**Batch (32 codes) 距离估计**——对单 query + 32 个 data vector：

直接复用 [PQ4xfs FastScan SIMD impl](./product-quantization.md)：把 D-bit string 每 4-bit 切片为 LUT，与 PQ4xfs 相同的 SIMD shuffle 流程。**comparable 速度**（同 PQ4xfs FastScan）但 RaBitQ 用 D bits 即 PQ 的一半 → **same throughput + 一半 storage + 更准 distance**。

### 5. Error-bound-based re-ranking for ANN（§4）

[gao-2024-rabitq §4]

与 IVF 结合时（KMeans bucketing data vectors）：

```
Query: q_r
1. Find nearest N_probe IVF clusters
2. For each data vector x_i in selected clusters:
     est_dist_i = RaBitQ_estimate(q_r, x_i)
     lower_bound_i = est_dist_i - error_bound  （from Eq 14, w.h.p.）
3. Maintain top-K candidates with exact distance
4. Drop x_i if lower_bound_i > current k-th NN exact distance
5. Otherwise compute exact distance(q_r, x_i) and re-rank
```

**关键工程后果**：

| | PQ / OPQ / LSQ | **RaBitQ** |
|---|---|---|
| Re-ranking K' 调参 | exhaustive across datasets | **无需调参（ε₀ = 1.9 fixed）** |
| Re-ranking 触发条件 | 距离 estimate 排前 K' | **per-vector lower_bound 大于 current threshold** |
| Recall 在 PQ-failing 数据集 | MSong ≤60% even with rerank | **works**（由 error bound 保证） |
| 理论保证 | 无 | **probabilistic correctness w.h.p.** |

→ **drop-by-bound 替代 keep-top-K'**——是 quantizer 层的 K' 预测问题"dual 攻击"。

## 与 [TopK vs Iterator Model](../topics/topk-vs-iterator-model.md) 的关系（双层互补）

K' 预测问题在 wiki 内有**两条独立解决路径**——VBASE 在 query engine 层（[Iterator + RM](./relaxed-monotonicity.md)）；RaBitQ 在 quantization 层（error-bound rerank）：

```
                    K' 预测问题
                          │
            ┌─────────────┼─────────────┐
            ▼                            ▼
    Query engine layer             Distance estimator layer
    (VBASE 2023 OSDI)              (RaBitQ 2024 SIGMOD)
            │                            │
    Iterator + RM                  Per-vec lower bound
    动态 K̃ on-the-fly               drop if bound > threshold
            │                            │
    HNSW/IVFFlat/SPANN             IVF + RaBitQ
    全用 RM                         替代 PQ
```

**正交性**：两条路径攻击同一问题但作用层不同——理论上 **VBASE + RaBitQ 可叠加**：query engine 用 RM iterator 自适应 K̃，每个 IVF cluster 内部用 RaBitQ 的 error bound 做 quantization-level rerank。详见 [topics/topk-vs-iterator-model.md "K' 消除：双层路径"](../topics/topk-vs-iterator-model.md)。

## 与现有 quantizer 的对比

[gao-2024-rabitq Table 1]

| | RaBitQ (new) | PQ + variants (PQ / OPQ / LSQ / ScaNN) |
|---|---|---|
| Codebook | Randomly transformed bi-valued vectors | Cartesian product of sub-codebooks |
| Code length | **D bits** | 2D bits (default) |
| Distance estimator | **Unbiased + sharp error bound** | Biased, no error bound |
| Single impl | **Bitwise AND + popcount (★★)** | LUT lookup (★) |
| Batch impl | Fast SIMD (★★★) | Fast SIMD (★★★) |
| Re-ranking 调参 | **无** | exhaustive |
| 工业失败案例 | 6/6 datasets work | MSong / Word2Vec 灾难 |
| 集成 KMeans | ✗（不需要） | ✓（必需） |
| Index time GIST 960-d | 117s | 105s (PQ) / 291s (OPQ) / **>24h (LSQ timeout)** |

## 与 LSH / SRP 的对比（§6）

[gao-2024-rabitq §6 + ref 11, 22, 46, 51]

**LSH (Locality-Sensitive Hashing)**：保证 "返回某 distance ≤ (1+c)r 的向量 w.h.p."——但**不保证 per-vector distance estimation 误差**。所以 LSH 无法用于 RaBitQ 那种 error-bound rerank。

**SRP (Signed Random Projection)**：用类似 random projection + binarization 编码 angular value（不是 inner product）。三个差异：
1. SRP 估计 angular，RaBitQ 估计 inner product（squared distance）——**问题层面不同**
2. SRP 只 bound variance，**不 bound per-instance error**——无法 rerank
3. SRP 同时 binarize data + query；RaBitQ data 用 ±1/√D bi-valued，query 用 4-bit unsigned int——**误差仅来自 data 侧**（quantize query 误差用 Theorem 3.3 控制到 negligible）

→ RaBitQ 是 quantization 范畴；LSH/SRP 是 hashing 范畴。两者表面像（都用 random projection + binary code），但 guarantee 形式与适用场景根本不同。

## 与 wiki 已 ingest concept 的关系

### 与 [Product Quantization](./product-quantization.md) 的关系

RaBitQ 不是 PQ variant——是 **PQ replacement**：

| 维度 | PQ | RaBitQ |
|---|---|---|
| Codebook 构造 | KMeans on sub-segments | 几何（hypercube + random rotation） |
| Train 阶段需要 | 是 | **否** |
| Code 是什么 | M sub-codes × k bits | D-bit string |
| Distance | LUT lookup | bitwise AND + popcount |
| Theoretical guarantee | 无 | **O(1/√D) w.h.p.** |

→ RaBitQ 在 PQ-family 演化路径上是 **paradigm shift**——之前 OPQ / RQ / LSQ / ScaNN / VGPQ 都仍在 PQ "Cartesian product of sub-codebooks" 框架内调；RaBitQ 跳出该框架。

### 与 [ScaNN](./scann.md) 的关系

ScaNN [guo-2019-scann] 提出 anisotropic loss——score-aware 修改 PQ codebook training objective。**仍是 PQ family 内**。

[gao-2024-rabitq §5.1 footnote 6]：ScaNN 在 in-memory ANN 的优势主要源自 SIMD-based fast impl ([4, 5] = PQ4xfs by André et al.)。当 PQ 用同样 SIMD 时，**ScaNN 的优势消失**——所以 RaBitQ 论文把 ScaNN 排除在 baseline 外（用 OPQ4xfs 代表 PQ-family SIMD 最强）。

→ RaBitQ vs ScaNN：前者用 theoretical bound 取代 score-aware loss；两条路线对"PQ 系误差"问题的不同应对。

### 与 [VGPQ](./vgpq.md) 的关系

VGPQ [wei-2020-analyticdb-v] 在 PQ 上加 Voronoi 几何剪枝——还在 PQ 框架内 + 加 partition 优化。**与 RaBitQ 正交**：理论上可以 VGPQ codebook + RaBitQ-style theoretical analysis（未实证）。

### 与 [HNSW](./hnsw.md) / graph-based 的关系

[gao-2024-rabitq §4]：RaBitQ 论文与 graph-based 方法（HNSW/NGT-QG）的集成"would require much more efforts"——graph search 是 vector-by-vector greedy traversal，难批量化（PQ4xfs FastScan SIMD 也面临同样问题；NGT-QG 在 graph node 内嵌量化代码部分应对）。RaBitQ 主要与 IVF 结合实证。

## 实验设置与实测亮点

[gao-2024-rabitq §5]

### 数据集（6 dataset，million scale）

| Dataset | Size | D | Type |
|---|---|---|---|
| MSong | 992,272 | 420 | Audio |
| SIFT | 1,000,000 | 128 | Image |
| DEEP | 1,000,000 | 256 | Image |
| Word2Vec | 1,000,000 | 300 | Text |
| GIST | 1,000,000 | 960 | Image |
| Image | 2,340,373 | 150 | Image |

### 关键实测（Distance estimation accuracy）

[Fig 3] Time-Accuracy trade-off：
- RaBitQ-batch / RaBitQ-single 在所有 6 数据集 dominate OPQ4fs / PQ4fs / LSQ4fs
- MSong 上 PQ4xfs / OPQ4xfs **average rel error >100%**；RaBitQ <40%（max rel error）
- Word2Vec 上 OPQ4xfs **max rel error >200%**；RaBitQ ~75% max
- 默认 setting：RaBitQ D bits vs PQ/OPQ 2D bits → RaBitQ **同精度同时压缩 2×**

### 关键实测（ANN search）

[Fig 4 / Fig 10]：
- 6/6 dataset RaBitQ + IVF + error-bound rerank consistently 优于 OPQx4fs + IVF + 调参 rerank
- vs HNSW：6/6 dataset RaBitQ 优于 HNSW（HNSW 是 in-memory state-of-the-art baseline）
- **同 ε₀ = 1.9 + B_q = 4 across all 6 datasets**——OPQ 需 per-dataset rerank K' 调参（500/1000/2500 全跑过）

### 关键实测（无调参属性）

[Fig 5 + Fig 6]：
- ε₀ 从 0 → 4，recall 单调升 + 在 ε₀ ≈ 1.9 饱和——参数物理意义清晰
- B_q 从 1 → 8，error 单调降 + 在 B_q = 4 饱和——B_q = 1 (binary query) 误差太大说明纯 binary hashing 不够

详见 [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)。

## 工程实现要点

[gao-2024-rabitq §3, §5.1]

- 不实际 materialize codebook C_rand——只存 P 矩阵 + D-bit string
- Bitwise AND + popcount 用 AVX2 / AVX512 native instruction
- Index phase = (1) normalize, (2) sample P, (3) compute D-bit codes, (4) precompute ⟨ō,o⟩
- Query phase = (1) inverse transform q via P^{-1}, (2) randomized uniform scalar quantize to B_q bits, (3) bitwise AND + popcount + accumulate

**论文未提供 incremental insertion**——RaBitQ index 是否支持增量加点？理论上 yes（与 IVF 同），但 P 矩阵一旦采样必须 fixed（重 sample 会改变所有 codes）；实际工程需 vendor 实现 incremental append。

C++ 源码：[github.com/gaoj0017/RaBitQ](https://github.com/gaoj0017/RaBitQ)（已迁移至 [VectorDB-NTU/RaBitQ-Library](https://github.com/VectorDB-NTU/RaBitQ-Library)）。

## Open Questions

- **RaBitQ + graph-based 索引**：[gao-2024 §4] 明示 "applying our quantization method in graph-based methods... we leave it as future work"——HNSW/Vamana + RaBitQ 的工程细节（greedy traversal 的 vector-by-vector 距离与 batch SIMD 矛盾）开放
- **RaBitQ + DiskANN（Vamana on SSD）**：DiskANN 用 PQ 在 DRAM 做 navigation——RaBitQ 替换 PQ 后理论上更紧（D bits vs DiskANN PQ ~32 bytes）+ 更准；wiki 未实证
- **RaBitQ + SPANN**：SPANN 不用 quantization（centroids in DRAM + full posting on SSD）；RaBitQ 在 posting list 上能否减少 SSD 占用？理论上 yes，但 SPANN 论文反对量化（recall 上限不被量化限制）——trade-off 待研究
- **RaBitQ + [VBASE](../systems/vbase.md) iterator + RM**：理论上叠加可行（VBASE engine 用 RM iterator + IVF + RaBitQ rerank）；论文未涉及，VBASE 仅集成 IVFFlat/HNSW/SPANN 全精度
- **Incremental insertion / update 行为**：P 矩阵 fixed 后增量数据用同 P；但 normalization 用 cluster centroid——增量数据 cluster 漂移如何处理？[gao-2024 §3.1.1] 未深入
- **跨 model embedding 升级**：与 wiki 全 frontier 一致——quantization layer 同样需要重新建 index；RaBitQ 也不解决
- **更大 D 时的 hypercube 退化**：D=10000 时 |C|=2^10000 远超物理可枚举；论文未实测 D > 1000；理论上随 D 增大 1/√D 越小（更准），但 P 采样 + storage 成本如何 scale？
- **量化 query 的 B_q 自适应**：B_q = 4 across 6 datasets 但 D=128 vs 960 跨 7.5×——是否真的 universal？[Theorem 3.3] B_q = Θ(log log D) 表明对极大 D 仍 valid，但实测仅到 D=960
- **与 [ScaNN anisotropic loss](./scann.md) 联合**：RaBitQ codebook 是 distribution-uniform；ScaNN 是 score-aware。理论上是否可以 hybrid（先 RaBitQ 找候选 + ScaNN 精排）？未探索
- **ε₀ = 1.9 vs 不同 use case**：要求 99.9% recall 时 ε₀ 应增大；要求低延迟时 ε₀ 应减小——但论文实测固定 1.9 在 6/6 数据集 work——这是否是因为 6 数据集 distribution 接近 normal？outlier-heavy 数据集行为未实测

Cited by: 待 query 引用
