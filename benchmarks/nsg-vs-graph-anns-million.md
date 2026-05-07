---
title: NSG vs Graph ANNs on Million-Scale (SIFT / GIST / RAND / GAUSS)
type: benchmark
sources: [fu-2017-nsg]
related: [../concepts/nsg.md, ../concepts/hnsw.md]
created: 2026-05-07
updated: 2026-05-07
---

# NSG vs Graph ANNs on Million-Scale

**TL;DR**: NSG 论文 §4.1 在四个百万级数据集上系统对比 NSG / HNSW / FANNG / Efanna / KGraph / DPG。NSG 在所有数据集 + 高 precision 区间击败 HNSW；内存仅为 HNSW 的约 1/3；索引时间介于 HNSW 与重型 graph 算法之间。[fu-2017-nsg §4.1 + Table 2/3 + Fig 6]

## 实验设置

- **数据集**：
  - SIFT1M：128 维 SIFT，1M 向量，LID=12.9，10k queries
  - GIST1M：960 维 GIST，1M 向量，LID=29.1，1000 queries
  - RAND4M：128 维均匀分布，4M 向量，LID=49.5
  - GAUSS5M：128 维高斯分布，5M 向量，LID=48.1
- **对手**：[HNSW](../concepts/hnsw.md)、FANNG、Efanna、KGraph、DPG（图）+ Flann、Annoy、FALCONN、[Faiss IVFPQ](../concepts/product-quantization.md)、Serial-Scan（非图基线）
- **方法**：单线程查询，8 线程构建。1% 验证集上 grid-search 调参后再测全集。
- **硬件**：i7-4790K（SIFT/GIST），Xeon E5-2630（RAND/GAUSS），96 GB RAM

[fu-2017-nsg §4.1.1 + Table 1]

## 索引指标对比（Table 2）

| 数据集 | 算法 | 内存 (MB) | AOD | MOD | NN(%) |
|---|---|---|---|---|---|
| SIFT1M | **NSG** | **153** | 25.9 | 70 | 99.3 |
| | HNSW₀（仅底层） | 451 | 32.1 | 50 | 47.5 |
| | FANNG | 374 | 30.2 | 98 | 60.4 |
| | Efanna | 1403 | 300 | 300 | 99.4 |
| | KGraph | 1144 | 300 | 300 | 99.4 |
| | DPG | 632 | 163.1 | 1260 | 99.4 |
| GIST1M | **NSG** | **267** | 26.3 | 70 | 98.1 |
| | HNSW₀ | 667 | 23.9 | 70 | 47.5 |
| | KGraph | 2154 | 400 | 400 | 98.1 |
| RAND4M | **NSG** | **2700** | 174.0 | 220 | 96.4 |
| | HNSW₀ | 6700 | 161.0 | 220 | 76.5 |
| GAUSS5M | **NSG** | **2600** | 146.2 | 220 | 94.3 |
| | HNSW₀ | 6700 | 131.9 | 220 | 57.6 |

注：
- AOD = average out-degree, MOD = max out-degree
- NN(%) = 节点链到其真实最近邻的比例
- HNSW 底层 NN% 偏低，因为最近邻被分配到上层；NSG 单层、所有最近邻都在同一图。

观察：
- NSG 在所有数据集上**内存最小**（约为 HNSW 1/3）。
- NSG 的 MOD 真正受限（70–220）；KGraph/Efanna 把 MOD 直接定到 300+，吃内存。
- NSG 比 FANNG 更小、NN% 更高（99.3 vs 60.4），说明 MRNG 近似比 RNG 近似更精确。

## 索引时间（Table 3）

| 数据集 | NSG (kNN + Alg 2) | HNSW | KGraph | FANNG | DPG |
|---|---|---|---|---|---|
| SIFT1M | 140 + 134 s | 376 s | 824 s | 1860 s | 1120 s |
| GIST1M | 1982 + 2078 s | 4010 s | 4300 s | 34540 s | 6700 s |
| RAND4M | (2.1 + 2.5) h | 5.6 h | 4.9 h | 38.3 h | 6.0 h |
| GAUSS5M | (2.3 + 2.5) h | 6.7 h | 5.1 h | 46.1 h | 6.4 h |

- NSG 比 HNSW 在 SIFT1M 上略慢（约 +30%），其他三个数据集上更快或持平。
- NSG 比 FANNG / DPG 快 1+ 数量级。

## 查询性能（Fig 6, 高 precision 区间）

精确数字论文是图，定性结论：
- **NSG 在所有四个数据集 + 0.9–0.999 precision 区间内 QPS 排第一**。
- HNSW 第二，与 NSG 差距随 precision 升高扩大。
- KGraph / Efanna 在 0.9 以下能竞争，但 0.95+ 急剧下降。
- DPG / FANNG 全程被压制。
- **LID 越高（GIST > RAND > SIFT），NSG 的相对优势越大**。

## 连通性（Table 4 in appendix）

| 数据集 | NSG SCC | HNSW SCC | FANNG SCC | KGraph SCC | DPG SCC |
|---|---|---|---|---|---|
| SIFT1M | 1 | 1 | 1 | 1 | 1 |
| GIST1M | 1 | 1 | 5 | 5 | 6 |
| RAND4M | 1 | 1 | 30 | 27 | 33 |
| GAUSS5M | 1 | 1 | 44 | 36 | 41 |

只有 NSG 和 HNSW 在所有数据集上保证连通（SCC=1）。FANNG / KGraph / DPG 在更难的数据集上断成几十个连通分量 —— 这是它们在高 precision 区间崩盘的根因。

## 可信度评估

- **实验设计**：作者提出方，但参与对比的所有算法都用了开源实现 + 公开调参流程；ann-benchmarks 等独立测试也支持 HNSW / NSG 在第一梯队的结论。
- **潜在偏向**：高 precision 区间是 NSG 优势区，论文 Fig 6 把 x 轴重点拉到 0.88+；中低 precision 下 KGraph / Efanna 实际更快，论文承认但不强调。
- **复现难度**：低。代码 + 数据集均公开（[ZJULearning/nsg](https://github.com/ZJULearning/nsg)、[BIGANN / SIFT](http://corpus-texmex.irisa.fr/)）。
- **场景局限**：百万级、单线程、L2 距离、稠密向量。十亿级、非欧氏、稀疏数据未在本节评估（DEEP100M 在 §4.2、Taobao 2B 在 §4.3 单独评估）。
