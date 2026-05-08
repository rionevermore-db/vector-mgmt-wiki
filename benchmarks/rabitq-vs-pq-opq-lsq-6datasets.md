---
title: RaBitQ vs PQ / OPQ / LSQ on 6 Datasets（Distance Estimation + ANN Search）
type: benchmark
sources: [gao-2024-rabitq]
related: [../concepts/rabitq.md, ../concepts/product-quantization.md, ../concepts/scann.md, ../concepts/vgpq.md, ../concepts/hnsw.md, ../systems/faiss.md, ../systems/diskann.md, ../topics/index-selection.md, ../topics/topk-vs-iterator-model.md]
created: 2026-05-08
updated: 2026-05-08
---

# RaBitQ vs PQ / OPQ / LSQ on 6 Datasets

**TL;DR**: RaBitQ SIGMOD 2024 论文 §5 系统对比 RaBitQ vs **PQ / OPQ / LSQ + Faiss FastScan SIMD impl** 在 6 个工业数据集上 (1) distance estimation accuracy, (2) ANN search time-recall tradeoff, (3) ablation 参数稳定性。**核心发现**：RaBitQ 在所有 6 数据集上**用一半 code length**（D bits vs PQ default 2D bits）**仍 dominate** PQ/OPQ/LSQ time-accuracy 曲线；在 PQ 灾难性失败的数据集（MSong / Word2Vec）RaBitQ 仍稳定 work；**ε₀ = 1.9 + B_q = 4 cross all datasets 完全无需调参**——这是与 PQ-family "exhaustive K' / hyperparameter tuning per dataset" 的根本差异。

## 实验设置

[gao-2024-rabitq §5.1]

### 数据集

| Dataset | Size | D | Type | 备注 |
|---|---|---|---|---|
| Msong | 992,272 | 420 | Audio | PQ baseline 灾难（rel error >100%） |
| SIFT | 1,000,000 | 128 | Image | PQ baseline 工作良好 |
| DEEP | 1,000,000 | 256 | Image | PQ baseline 工作良好 |
| Word2Vec | 1,000,000 | 300 | Text | PQ baseline 灾难（max rel error >200%） |
| GIST | 1,000,000 | 960 | Image | 高维 PQ baseline 良好 |
| Image | 2,340,373 | 150 | Image | 最大 dataset |

数据集来源：[ann-benchmarks](http://ann-benchmarks.com/) + [big-ann-benchmarks](https://big-ann-benchmarks.com/)。

### 硬件

- CPU: AMD Threadripper PRO 3955WX 3.9 GHz, Zen2, AVX2 SIMD
- RAM: 64 GB
- OS: Ubuntu 20.04 LTS
- Compiler: g++ 9.4.0 with `-Ofast -march=core-avx2`
- Python: 3.8 driver

### Baseline 实现

[§5.1] 全部用 Faiss 1.7.4 release（well-optimized AVX2 SIMD）：

| Method | 实现 | 备注 |
|---|---|---|
| **PQ** | Faiss IVF_PQ4xfs / PQx8 | jegou-2011-pq baseline |
| **OPQ** | Faiss IVF_OPQ4xfs / OPQx8 | rotated PQ |
| **LSQ** | Faiss IVF_LSQx4fs | additive quantization (Martinez 2018) |
| **HNSW** | hnswlib + AVX2 | M=16, ef_construction=500, ef_search varied |
| **RaBitQ** | C++17 (gaoj0017/RaBitQ) | 论文作者实现 |
| **ScaNN** | 排除 | §5.1 footnote 6: ScaNN 优势主要源自 PQ4xfs FastScan SIMD；同 SIMD 时 ScaNN 优势消失 |

### 参数设置

[§5.1, §5.2.4-5.2.5]

| Parameter | RaBitQ | PQ / OPQ / LSQ |
|---|---|---|
| ε₀ (rerank confidence) | **1.9 fixed** | n/a |
| B_q (query quantization bits) | **4 fixed** | n/a |
| M (sub-codebook count) | n/a | D/2 (固定) |
| Re-rank candidates | **不需要**（drop by bound） | 500 / 1000 / 2500 三档全测 |
| IVF clusters | 4096 (Faiss recommend) | 4096 |
| Code length | **D bits（pad to 64-multiple）** | 2D bits default |

→ **RaBitQ 有 2 个参数（ε₀, B_q）**but **跨 6 数据集均 fixed**；PQ/OPQ/LSQ 有 (M, k, K_rerank) 至少 3 个调参 + 每数据集需独立 tune。

## 主结果

### Result 1: Time-Accuracy Trade-Off for Distance Estimation（Fig 3）

[§5.2.1]

每行展示 average rel error 与 max rel error vs time per vector（ns）。"x4fs-batch" = 4-bit code SIMD batch；"x8-single" = 8-bit code single vector LUT；"RaBitQ-batch / -single" = RaBitQ 两种 impl。

**核心观察**：

1. **RaBitQ-batch / RaBitQ-single 在所有 6 dataset dominate** OPQ4fs / PQ4fs / LSQ4fs 曲线（曲线左下 = 更优）

2. **Default setting 下 RaBitQ 用一半 code length 仍更准**：
   - RaBitQ default = D bits
   - PQ/OPQ default = 2D bits
   - 同精度时 RaBitQ 节省 50% storage

3. **MSong PQ 灾难**：
   - PQx8 / OPQx8 average rel error **>100%**
   - PQ4xfs / OPQ4xfs average rel error **>100%**
   - RaBitQ <40% (max rel error)

4. **Word2Vec PQ 灾难**：
   - OPQ max rel error **>200%**
   - RaBitQ ~75% (max)

5. **SIFT/DEEP/GIST**：PQ/OPQ 工作良好，**RaBitQ 仍系统优于但优势缩窄**

### Result 2: Indexing Time（Table 4, GIST 960-d）

[§5.2.2]

| Method | Index time | k = sub-codebook bits |
|---|---|---|
| RaBitQ | **117s** | n/a |
| PQ | 105s | k=4 |
| OPQ | 291s | k=4 |
| LSQ | **>24h timeout** | k=4 |

→ RaBitQ 与 PQ 同量级（117s vs 105s），无 KMeans 训练成本；OPQ rotation 训练 2× 慢；LSQ NP-hard 优化 unscalable。

### Result 3: ANN Search Time-Accuracy（Fig 4）

[§5.2.3]

QPS vs Recall 曲线 + QPS vs avg distance ratio。

| Dataset | RaBitQ best vs OPQx4fs best | RaBitQ vs HNSW |
|---|---|---|
| Image | dominates OPQ4fs at all 3 rerank settings | dominates HNSW |
| GIST | dominates OPQ | dominates HNSW |
| Msong | OPQ recall **abnormally drops** 当 IVF 探更多 buckets | RaBitQ stable, dominates HNSW |
| SIFT | OPQx4fs comparable | RaBitQ ≈ HNSW |
| DEEP | OPQx4fs comparable | RaBitQ ≈ HNSW |
| Word2Vec | OPQ 不可用 | RaBitQ dominates |

**关键观察**：

- **OPQ 在 Msong 上"探更多 buckets recall 反而降"**——错误 distance estimate 让 rerank candidate 排序错位；rerank K'=2500 大几乎全扫仍不修正。
- **RaBitQ 没有这种 pathology**——error bound 保证排序错位的概率受控
- **RaBitQ 6/6 dataset dominate HNSW**（Faiss + RaBitQ + IVF rerank vs hnswlib）

### Result 4: ε₀ Verification（Fig 5）

[§5.2.4]

Recall vs ε₀ on SIFT (D=128) and GIST (D=960)：

| ε₀ | SIFT recall | GIST recall |
|---|---|---|
| 0 | low | low |
| 1.5 | ~95% | ~93% |
| **1.9** | **~99%** | **~99%** |
| 2.5 | ~99.9% | ~99.9% |
| 4 | ~99.99% | ~99.99% |

→ **ε₀ ≈ 1.9 是 wiki-side 最优 plateau**——固定值跨数据集。这是 RaBitQ "无调参"属性的具体证据。

### Result 5: B_q Verification（Fig 6）

[§5.2.5]

Average rel error vs B_q on SIFT (D=128) and GIST (D=960)：

| B_q | SIFT avg rel error | GIST avg rel error |
|---|---|---|
| 1 (binary) | **~15%** | **~15%** |
| 2 | ~5% | ~5% |
| **4** | **<2%** | **<2%** |
| 8 | <1% | <1% |

→ B_q = 4 收敛到误差饱和；B_q = 1 (pure binary) **误差大 5×+**——这解释为什么 binary hashing 方法（LSH / SRP）无法获得高精度。

### Result 6: Unbiasedness Verification（Fig 7）

[§5.2.6]

Linear regression on (true squared distance, estimated squared distance) for 10^7 pairs from GIST：

| Estimator | Slope (regression) | Intercept | 偏差 |
|---|---|---|---|
| RaBitQ ⟨ō,q⟩/⟨ō,o⟩ | **1.00** | 0.00 | **unbiased** |
| OPQx4fs (PQ4xfs) | 0.72 | 0.02 | **clearly biased** |

→ Empirically 验证 [Theorem 3.2] 的 unbiasedness。

## Ablation Studies（Appendix F）

[gao-2024-rabitq §F.1-F.3]

### F.1: Codebook Construction Ablation（Table 6, GIST）

| Codebook | Avg Rel Error | Max Rel Error |
|---|---|---|
| Random (RaBitQ) | **1.675%** | **13.043%** |
| Learned (PQ KMeans) | 3.049% | 34.375% |

→ **Random codebook 优于 learned**——反直觉但论文解释：PQ 的 search space 极大（K-means cluster on subcontiguous segments），heuristic 学习给出 suboptimal solution；RaBitQ 的 random codebook **由理论保证均匀** + Lipschitz concentration → 比 learned 更稳。

### F.2: Estimator Ablation（Table 7, GIST）

| Estimator | Avg Rel Error | Max Rel Error |
|---|---|---|
| RaBitQ ⟨ō,q⟩/⟨ō,o⟩ | **1.675%** | **13.043%** |
| Treat ō as data (PQ-style) | 2.196% | 52.400% |

→ 把 quantized vector 当作 data vector（PQ 的做法）误差大 4×（max rel error 52% vs RaBitQ 13%）——这是为什么 PQ 没 error bound 的根因。

### F.3: Re-Ranking Ablation（Fig 10）

with vs w/o re-ranking on 6 datasets：

→ **Re-ranking 必需**——RaBitQ 的 distance estimate accurate 但绝对距离接近时仍需精排来防止 ranking 错位。

## 可信度评估

### 实验设计

- ✓ 数据集 + 索引参数完整披露
- ✓ 所有 baseline 用同 Faiss release（统一 SIMD impl）
- ✓ Open source 代码 + 数据
- ✓ 6 dataset 跨 audio/image/text/multiple D
- ✓ Theoretical bound + empirical 全部一致

### 复现难度

- RaBitQ C++ 开源 (gaoj0017/RaBitQ)
- Baseline 全部 Faiss release（公开）
- Hardware 是消费级 Threadripper（非企业 server）→ 易复现
- ann-benchmarks / big-ann-benchmarks dataset 公开

### 偏向

- **作者论文自评**——baseline 选择对作者方法 favorable 是潜在风险
- 但 ablation 充分 + theoretical 与 empirical 一致 + 开源代码可独立验证
- ScaNN 被排除是合理（footnote 6 解释清楚 ScaNN 优势源自 SIMD impl 而非 algorithm）
- HNSW 作为 reference 充分

### 数据集偏向

- 6 dataset 全 ≤ 1M（million-scale）
- 缺 billion-scale（SIFT1B / DEEP1B / SPACEV1B）
- 缺真实 production 数据（Bing / 商业搜索）

## Open Questions

- **Billion-scale 实证**：6 dataset 全 million，未测 1B+。当 IVF cluster 数 ↑ + 单 cluster 内 vector ↓ 时 RaBitQ codebook quality 是否仍 hold？
- **Distributed RaBitQ**：单实例 in-memory，未实证多机
- **GPU impl**：Bitwise + popcount 在 GPU 是否仍快于 PQ LUT？(GPU LUT 受 register 限制 [WarpSelect](../concepts/warpselect.md))，未实测
- **真实 query distribution（非 uniform）**：所有实验 query 来自 dataset 自带 query set——production query 可能 OOD
- **RaBitQ + filter / multi-vector / range 复杂 query**：未测 [topics/topk-vs-iterator-model.md] 涵盖的 Q4-Q8 复杂查询
- **极高维 D > 1000 (LLM embedding)**：现代 OpenAI ada-002 是 1536-d；GIST 960-d 接近上限但未达；wiki 未覆盖
- **Sparse vector 量化**：所有 dataset 是 dense；sparse vector / SPLADE-style 量化未测
