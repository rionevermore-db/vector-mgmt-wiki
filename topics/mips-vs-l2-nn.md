---
title: MIPS vs L2-NN（最大内积搜索 vs 最近邻搜索）
type: topic
sources: [guo-2019-scann, malkov-2016-hnsw, jegou-2011-pq, fu-2017-nsg]
related: [../concepts/scann.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/proximity-graph.md]
created: 2026-05-07
updated: 2026-05-07
---

# MIPS vs L2-NN

**TL;DR**: ANN 文献长期把"近邻"等同于"L2 距离最小"。但现代推荐系统、softmax 近似、extreme classification 实际做的是 **Maximum Inner Product Search (MIPS)**：找 `argmax_x <q,x>`。MIPS 与 L2-NN **不等价**——向量范数会影响排序。这一区别决定了量化 loss 的设计、graph entry point 的选择，乃至能否用 normalize 后跑 L2-NN 算法的常见 trick。[guo-2019-scann §1-2]

## 问题陈述

- **L2-NN**：`x* = argmin_x ||q − x||²`
- **MIPS**：`x* = argmax_x <q, x>`

展开 L2 平方：`||q−x||² = ||q||² + ||x||² − 2<q,x>`

- 给定 q，`||q||²` 是常数，不影响排序
- 但 `||x||²` 不是常数 —— **范数大的 x 在 L2-NN 下天然吃亏**，但在 MIPS 下天然占优

**结论**：L2-NN 与 MIPS 仅当所有 x 同范数时等价（典型例：cosine similarity = 单位向量上的内积，与单位向量上的 L2-NN 等价）。

## 这个区别在哪些场景下重要

| 场景 | 任务 | 范数差异来源 |
|---|---|---|
| 推荐系统（双塔） | MIPS | 物品热度训练成范数 |
| Softmax 近似 / extreme classification | MIPS | 类权重 row 范数与频率相关 |
| 图像 / 文档检索（cosine） | L2-NN ≡ MIPS（normalize） | 训练时显式归一化 |
| BIGANN / SIFT 类视觉 ANN benchmark | L2-NN | 描述符未归一 |
| Embedding 检索（OpenAI / sentence-transformers 默认） | cosine ≡ MIPS | normalize 是惯例 |

[guo-2019-scann §1, §2.1]

## 工业方案对比

| 算法 | 原生支持 | MIPS 支持方式 | 关键 trade-off |
|---|---|---|---|
| [HNSW](../concepts/hnsw.md) | L2 / cos | 直接换距离函数；entry point 仍随机选 | 在 MIPS high-recall 区被 [ScaNN](../concepts/scann.md) 击败 [guo-2019-scann Fig 4b] |
| [NSG](../concepts/nsg.md) | L2 | 同上；MRNG 理论是 L2 monotonicity，MIPS 下未验证 | 论文未涉及 MIPS 评估 |
| [PQ / IVFPQ](../concepts/product-quantization.md) | L2 reconstruction | ADC 用内积代替 L2 距离；codebook 仍按 L2 学 | 是 ScaNN 想替代的对象 |
| [ScaNN](../concepts/scann.md) | **MIPS 原生** | Score-aware loss 直接学 anisotropic codebook | L2-NN 任务下回归到 PQ |

## 常见的"Reduction"误区

**用 normalize + L2-NN 替代 MIPS 是错的**（除非数据本身已 unit norm）：

- Trick：把所有 x 加一维变 `(x, sqrt(R² − ||x||²))`，q 加一维变 `(q, 0)` —— 让 L2-NN 等价 MIPS（Bachrach 2014）。
- 实际：会引入额外维度，破坏数据结构；高维下"补维"几乎全是 0 / 接近 0，索引性能下降。
- ScaNN 论文走的是另一条路：**直接重新定义量化 loss**，而不是把 MIPS 转成 L2 问题。[guo-2019-scann §2.2.2]

## 对 wiki 现有 page 的影响

- [HNSW](../concepts/hnsw.md), [NSG](../concepts/nsg.md), [PQ](../concepts/product-quantization.md) 的复杂度证明、benchmark 数据、推荐参数 **都是 L2 语境**。MIPS 下需要单独验证。
- [ScaNN](../concepts/scann.md) 是 wiki 里目前唯一**原生 MIPS** 的算法。
- [proximity-graph.md](../concepts/proximity-graph.md) 的 MRNG / Delaunay monotonicity 理论是 L2 距离的 —— MIPS 下"单调路径"是否成立未知（开放问题）。

## Open Questions

- **MIPS 下的 Delaunay 类对应物**？L2-Delaunay 保证朴素贪心找最近邻；MIPS 是否有等价结构？尚未在 wiki 已 ingest 的文献里看到。
- **Graph methods 在 MIPS 下的理论 gap**：[HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md) 的 log 复杂度证明假设 L2 度量。MIPS 不是 metric（不满足三角不等式），证明不直接迁移。
- **混合检索**：同一服务既要 MIPS（基于内积的语义召回）又要 L2-NN（基于嵌入距离的去重）—— 共享一个 index 还是两套？工业实践（Pinecone / Vespa / Milvus）的处理方式 wiki 未覆盖。
