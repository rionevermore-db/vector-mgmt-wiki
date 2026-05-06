---
title: PQ on SIFT / GIST recall + 2B SIFT 可扩展性
type: benchmark
sources: [jegou-2011-pq]
related: [../concepts/product-quantization.md]
created: 2026-05-07
updated: 2026-05-07
---

# PQ on SIFT / GIST recall + 2B SIFT 可扩展性

**TL;DR**: PQ 论文 §V 的核心实验。证明 ADC 在 SIFT / GIST 上完胜 spectral hashing 与 Hamming embedding，IVFADC 比 ADC 快 1–2 个数量级且内存比 FLANN 小一个数量级，PQ 路径可扩展到 2B SIFT 描述符。[jegou-2011-pq §V]

## 实验设置

- **数据集**：
  - SIFT 1M：128 维，1M database + 100k learning + 10k queries（INRIA Holidays + Flickr1M）
  - GIST 1M：960 维，1M database + 100k learning + 500 queries
  - 2B SIFT：100 万图像 × ~2k 描述符
- **指标**：recall@R（query 的真实最近邻是否在前 R 个返回结果里）；R=1 即 precision。
- **PQ 默认参数**：m=8, k*=256（即 64 bit code）
- **基线**：spectral hashing (SH)、Hamming embedding (HE)、FLANN
- **硬件**：单核

[jegou-2011-pq §V.A + Table III]

## 结果

### 内存 vs 准确率（SIFT 1M）

m=8, k*=256（64 bit）下 ADC recall@100 ≈ 0.65。增大 m 或 k*（更长 code）持续提升召回；m=16, k*=4096 接近召回 1.0 但 code length 已到 192 bit。[jegou-2011-pq §V.B + Fig 6]

### IVFADC 时序（GIST 1M, 500 queries, 64-bit code）

| 方法 | 参数 | 查询时间 | 扫描的 code 数 | recall@100 |
|---|---|---|---|---|
| SDC | — | 16.8 ms | 1,000,991 | 0.446 |
| ADC | — | 17.2 ms | 1,000,991 | 0.652 |
| IVFADC | k'=1024, w=1 | **1.5 ms** | 1,947 | 0.308 |
| IVFADC | k'=1024, w=8 | 8.8 ms | 27,818 | 0.682 |
| IVFADC | k'=1024, w=64 | 65.9 ms | 101,158 | 0.744 |
| IVFADC | k'=8192, w=8 | 10.2 ms | 2,709 | 0.516 |
| Spectral hashing | — | 22.7 ms | 1,000,991 | 0.132 |

[jegou-2011-pq Table V]

关键观察：
- IVFADC k'=1024, w=8 比 ADC 快近 2× 且召回更高（0.682 vs 0.652）。
- SH 任意配置下都被 PQ 完爆。
- w 是召回-速度调节钮，随 w 单调升。

### vs FLANN

- 多数 operating point 上 IVFADC 召回更高、查询更快。
- 内存：IVFADC 索引 <25 MB；FLANN >250 MB（且需原向量驻留做 re-ranking）。[jegou-2011-pq §V.D + Fig 10]

### 2B SIFT 可扩展性

Fig 11：HE 和 IVFADC 单查询时间随 database size 从 10M 增到 2B 的曲线。两者都使用同样的 20k 词表 coarse quantizer 和 64 bit signature。
- 在 2B 规模下 IVFADC 单 vector 查询时间 ≈ 3.3 ms（HE 约 2.8 ms）。
- 整索引能装入 128 GB 单机（≈9 字节/向量）。[jegou-2011-pq §V.F + Fig 11]

### 维度分组的影响（SIFT, m=4, k*=256）

| Order | recall@100 |
|---|---|
| natural | 0.593 |
| random | 0.501 |
| structured（按 patch grouping） | **0.640** |

[jegou-2011-pq Table IV]

领域知识 → 大约 +5pp 召回提升。

## 可信度评估

- **实验设计**：作者提出方，但同时报告了 SDC / ADC / IVFADC 三种自家方法、与三个独立基线（SH、HE、FLANN）对比，多角度。
- **潜在偏向**：FLANN 对比里 PQ 用了 64 bit code，FLANN 用了 256 字节原向量 —— 内存差距 32×，比较有利于 PQ 的内存维度。但召回数字也确实更高，并非仅因内存。
- **复现难度**：低。SIFT / GIST 数据公开，作者代码与数据均开源（[INRIA texmex](http://www.irisa.fr/texmex/people/jegou/ann.php)）。
- **场景局限**：单核、单机、SIFT / GIST 描述符。GPU、分布式、现代 deep learning 嵌入向量（更高维、更稠密）下结论需重新验证 —— 这是 Faiss 后续工作的重点。
