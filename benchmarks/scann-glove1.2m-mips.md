---
title: ScaNN on Glove1.2M MIPS（Recall + Speed-Recall）
type: benchmark
sources: [guo-2019-scann]
related: [../concepts/scann.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../topics/mips-vs-l2-nn.md]
created: 2026-05-07
updated: 2026-05-07
---

# ScaNN on Glove1.2M MIPS

**TL;DR**: ScaNN 论文 §5 在 Glove1.2M 词嵌入上验证 anisotropic loss 优于 reconstruction loss（Recall1@10 0.91 vs 0.83 @ 200 bit），且击败 ann-benchmarks 上 11 个主流算法（包括 HNSW 与 faiss-IVF），尤其在高 recall 区。[guo-2019-scann §5 + Fig 3, 4]

## 实验设置

- **数据集**：Glove1.2M
  - 1.2 million GloVe 词嵌入
  - 100 维
  - 任务：cosine similarity（≡ MIPS over unit-normed vectors）
  - 1k queries
- **基线（quantization 类）**：reconstruction-loss PQ、LSQ、QUIPS-Cov(x)、QUIPS-Cov(q)、QUIPS-Opt
- **基线（ann-benchmarks 11 个）**：annoy, mrpt, hnsw(nmslib), hnsw(faiss), faiss-ivf, NGT-panng, NGT-onng, rpforest, kgraph, sw-graph(nmslib), Ours (ScaNN)
- **硬件**：Intel Xeon W-2135，**单线程**
- **指标**：Recall1@N（N 个返回结果中是否含真实 top-1）

[guo-2019-scann §5.1, 5.2, 5.3]

## 结果

### Direct loss 对比（Fig 3a, 200 bit）

| Loss | Recall1@10 |
|---|---|
| Traditional reconstruction | ≈ 0.83 |
| Score-aware (T=0.2) | **≈ 0.91** |
| Score-aware (T=0.5) | ≈ 0.85 |
| Score-aware (T=0.9) | ≈ 0.79（过度惩罚平行误差） |

观察：T 存在最优值（约 0.2），过大或过小都更差；论文按理论极限 T=0.2 → η=4.125 给出推荐配置。

### 内积估计精度（Fig 3b, top-1）

跨 50–400 bit 范围，score-aware 估计的相对误差 `|<q,x> − <q,x̃>| / <q,x>` 全程低于 reconstruction loss。

### vs quantization SoTA（Fig 4a, 100 bit / 200 bit）

ScaNN（Ours）的 Recall1@N 曲线在两种 bitrate 下都在最上方。100 bit 下与 LSQ 差距更大；200 bit 下差距收窄但仍胜。被压制的：QUIPS-Cov(x), Cov(q), Opt, LSQ。

### vs ann-benchmarks 11 算法（Fig 4b, Recall 10@10）

定性顺位（高 recall 区，~0.95 左右）：

| 排名 | 算法 | 备注 |
|---|---|---|
| 1 | **ScaNN** | ~5000 QPS @ recall 0.95 |
| 2 | hnsw(nmslib) | ~3000 QPS |
| 3 | NGT-onng | ~3000 QPS |
| 4 | hnsw(faiss) | ~2500 QPS |
| 5 | NGT-panng | ~2000 QPS |
| 6+ | faiss-ivf, kgraph, sw-graph, mrpt, annoy, rpforest | <2000 QPS |

随 recall 升至 0.99+，ScaNN 与 [HNSW](../concepts/hnsw.md) 的差距进一步扩大；recall <0.9 区间 HNSW 反而更快。

## 可信度评估

- **实验设计**：作者用 ann-benchmarks 公开协议，所有对比算法参数取自其官方调优结果。这是 ANN 社区认可的最公允基准。
- **潜在偏向**：
  1. 仅 Glove1.2M 一个数据集做最终对比；未在 SIFT1M / DEEP1M 等 L2-NN 标准 benchmark 上跑（因为 ScaNN 主打 MIPS，作者明确选了 cosine 数据集）。
  2. 单线程；多线程下 graph 算法吞吐扩展性可能不同。
  3. 256 bit 以上 ScaNN 优势收窄（Fig 3a 显示 T 选择更敏感）。
- **复现难度**：低。代码、数据、ann-benchmarks 配置均开源（[google-research/scann](https://github.com/google-research/google-research/tree/master/scann)）。
- **场景局限**：MIPS 任务、word embedding（100 维偏低）、单 query。**100M+ 规模、高维 deep embedding（768 / 1024 维）下 ScaNN 是否仍领先未在本论文验证** —— 后续 ScaNN 文档与 Google Research 博客有补充材料。
