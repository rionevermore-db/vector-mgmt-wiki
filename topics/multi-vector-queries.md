---
title: Multi-Vector Queries（多向量查询）
type: topic
sources: [wang-2021-milvus, zhang-2023-vbase]
related: [../systems/milvus.md, ../systems/pinecone.md, ../systems/vbase.md, ../concepts/product-quantization.md, ../concepts/relaxed-monotonicity.md, ./mips-vs-l2-nn.md, ./topk-vs-iterator-model.md, ../benchmarks/vbase-8queries-recipe1m.md]
created: 2026-05-07
updated: 2026-05-08 (VBASE)
---

# Multi-Vector Queries

**TL;DR**: 一个 entity 由**多个向量**描述（如人脸前/侧/姿态、recipe 文本+图像、商品多视角）的 top-k 查询。两条算法路线：**vector fusion** 仅适用于可分解相似度（内积），把多向量拼接 + 加权聚合用单次 ANN 解决，3.4-5.8× 快；**iterative merging** 基于 Fagin's NRA 通用方案，doubling k' 直到 top-k 确定。Milvus 论文是 wiki 内唯一覆盖此主题的 source。

## 问题陈述

每个 entity X 由 μ 个向量 `v₀, v₁, ..., v_{μ-1}` 描述（multi-modal 或 multi-aspect）。query Q 对应给出 μ 个向量 `q.v₀, q.v₁, ..., q.v_{μ-1}`。

最终相似度 = 在每个 v_i 上计算的相似度 f(X.v_i, Y.v_i) 经聚合函数 g 合成：

```
sim(X, Y) = g(f(X.v₀, Y.v₀), f(X.v₁, Y.v₁), ..., f(X.v_{μ-1}, Y.v_{μ-1}))
```

约束 g monotonically non-decreasing（保证 top-k 单调性，[Fagin 2001 NRA]）。常见 g：weighted sum、average、median、min、max。

## 真实场景

[wang-2021-milvus §4.2]：

- **智能视频监控**：摄像头同时捕获前脸/侧脸/姿态三个 vector → 联合查询
- **Recipe 检索**：文本描述 + 图像 → 联合检索菜谱
- **跨模态行人 ID**：multi-view 摄像头融合
- **同一对象多模型 embedding 融合**：BERT + RoBERTa 双 embedding 集成

## Naïve 解法

```
for each q.v_i:
  R_i ← top-k(q.v_i, D_i)   # 在 v_i 维度索引上独立 top-k
final ← top-k(g(...) over X | X ∈ ⋃ R_i)
```

问题：每个 R_i 独立取 k 个时 union 后**真实 top-k 可能完全不在内**（recall 实测 0.1）[wang-2021-milvus §4.2]。

## 算法 1：Vector Fusion（仅适用可分解相似度）

[wang-2021-milvus §4.2]

**前提**：相似度函数可分解，**典型例子是 inner product**：

```
g(<q.v₀, x.v₀>, <q.v₁, x.v₁>, ...) = <[w₀·q.v₀, w₁·q.v₁, ...], [x.v₀, x.v₁, ...]>
                                       └────── concat query ──────┘  └── concat data ──┘
```

→ 把 entity 多向量物理拼接成一个长向量，query 多向量加权拼接，**单次 ANN 解决**。

适用范围：
- ✓ Inner product（可分解）
- ✓ Cosine similarity（normalize 后等价 inner product，[per topics/mips-vs-l2-nn.md]）
- ✗ Euclidean distance（不可分解：‖q-x‖² ≠ Σ‖q.v_i - x.v_i‖²，仅有上界）
- ✗ Tanimoto / Jaccard（化学指纹 / 集合相似度）

[wang-2021-milvus Fig 16b] 实测内积场景：vector fusion 比 iterative merging **3.4×–5.8× 快**，因只需一次 ANN search。

## 算法 3：VBASE Iterator + NRA（绕开 doubling K'，2023 OSDI）

[zhang-2023-vbase §4.5]

VBASE 的方法是**用 iterator interface 代替 TopK**——多个 vector index 同时打开 iterator，**经典 NRA threshold-based termination**：

```
opened iterators: it_0, it_1, ..., it_{μ-1} (each on one vector column)
result_buffer = empty
loop:
  pick next iterator (greedy / round-robin) and i = it_i.amgetnext()
  add i to result_buffer
  threshold = sum(weight_j * it_j.current_distance) for all j
  if all iterators in Phase 2 (RM) AND result_buffer top-K's score > threshold:
    return top-K from result_buffer
```

**关键差异**：iterator interface 让 NRA 的"按距离顺序逐个推进"语义**真正可行**——RM 性质保证每个 iterator "再也不会更近"，可以安全用 threshold。这正是 [wang-2021-milvus §4.2] 提及的 "ANN 索引不支持 efficient `getNext()`" 障碍——**VBASE [zhang-2023-vbase] 通过 RM 形式化解决了这个问题**。

**Strategy 选择**：[Table 7] 实测 VBASE 自适应在 greedy / round-robin 间切换：

| Weight ratio | Round-Robin scans | Greedy scans / recall | VBASE scans / recall |
|---|---|---|---|
| 1:1 | 651.93 | 699 / 0.9313 (locally trapped) | **638.56 / 0.9705** |
| 1:5 | 463.39 | **372.96** / 0.9949 | 409.31 / **0.9961** |
| 1:10 | 363.47 | **274.86** / 0.9985 | 311.66 / **0.9987** |

→ Greedy 在权重悬殊时显著 reduce scans（推进当前最近 iterator，冷快），但权重均匀时陷局部最优；VBASE dynamic 切换提供两者的合并优势。

详见 [topics/topk-vs-iterator-model.md](./topk-vs-iterator-model.md) 与 [concepts/relaxed-monotonicity.md](../concepts/relaxed-monotonicity.md)。

## 算法 2：Iterative Merging（通用方案）

[wang-2021-milvus §4.2 Algorithm 2]

**前提**：仅要求 g monotonic（不要求 f 可分解）。基于 Fagin's NRA [wang-2021-milvus ref 19]。

**关键 challenge**：NRA 假设可 `getNext()` 拿到流式排序结果。**ANN 索引不支持 efficient `getNext()`**——只能跑完整 top-k' 查询。

**Milvus 改造**：
```
k' ← k                            # 起始候选规模
while k' < threshold:
  for i in 0..μ-1:
    R_i ← VectorQuery(q.v_i, D_i, k')   # 在每个 dim 上跑 top-k'
  if NRA stop condition on ⋃ R_i:
    return final top-k from ⋃ R_i
  else:
    k' ← k' × 2                   # doubling
return top-k from ⋃ R_i (best effort)
```

**两个优化超过原 NRA**：
1. 不依赖 `getNext()`，用 batch top-k' + doubling 替代——避免单点访问的代价
2. 引入 k' 上限阈值——防止退化场景下无限增长

[wang-2021-milvus Fig 16a] 实测 Euclidean 场景（Recipe1M, 文本+图像 multi-vector）：iterative merging k'=4096 比 NRA-2048 快 **15×** 且 recall 相当。

> **2023 VBASE 反例** [per zhang-2023-vbase §5.3 Q4-6 + benchmarks/vbase-8queries-recipe1m.md]：在 Recipe 330K × 1024-d × 双 vector column 上，**Iterative Merging 比 VBASE iterator 慢 200-300×**（Milvus Q4 99p 9300 ms vs VBASE 5.3 ms）。论文 §5.3 明示原因——"the algorithm tries different K' to produce a sufficiently large intersection of multiple TopK results... cannot finish after several rounds and **accumulates a large number of random reads**"。Iterative Merging 的 doubling K' 是 TopK 框架内的 best effort，但**根本上是因为 TopK 接口逼迫"先取 K' 个候选"**——在 multi-column 场景下"K' 翻倍 + merge"是几何级随机访问。

## 三种算法对比

| | Vector Fusion (2021 Milvus) | Iterative Merging (2021 Milvus) | **VBASE Iterator + NRA (2023)** |
|---|---|---|---|
| 相似度限制 | **必须可分解**（inner product / cosine） | 任意 monotonic g | **任意 monotonic g** |
| 接口前提 | 1 个 ANN topk(K) | μ × ANN topk(K') with doubling K' | **μ × iterator (Open/Next/Close + RM)** |
| ANN 调用次数 | **1 次** | μ × `log(k_final/k_init)` 次 | **μ × O(K) Next() 调用** |
| 速度（内积场景） | **3.4-5.8× 快**（Milvus 论文） | baseline | **Q4 比 Milvus iterative merging 200-300× 快**（VBASE Recipe1M 实测） |
| 速度（Euclidean） | ✗ 不可用 | 唯一 TopK-based 选择 | **唯一 iterator-based 选择** |
| 索引开销 | concat 后单一索引 | μ 个独立 index | **μ 个独立 index** |
| 增量更新 | concat index 需 rebuild | 各维度 index 独立更新 | **各维度 index 独立更新** |
| 是否需 K' 预测 | ✗（K = K_final） | **✓**（doubling K'） | **✗（RM 自动停）** |

## 与 wiki 现有概念的关联

- **L2 与 MIPS 的可分解性差异**——是这两条路线必须分叉的根因。详见 [topics/mips-vs-l2-nn.md](./mips-vs-l2-nn.md)。MIPS（内积）天然可分解 → vector fusion 的存在；L2 不可分解 → 必须 iterative merging。
- **[Product Quantization](../concepts/product-quantization.md) 在 multi-vector 下**：concat 后 PQ codebook 需为更高维 retrain；论文未深入。
- **[ScaNN](../concepts/scann.md) anisotropic loss 在 multi-vector 下**：MIPS 优化 + score-aware 在 fused vector 上是否仍有效？wiki 未覆盖。
- **[Attribute Filtering](./attribute-filtering.md) + Multi-vector**：两个 advanced query feature 的组合（"找与 q.v₀, q.v₁ 都相似且 price < 500"）论文未联合处理。

## Open Questions

- **新型 RAG 多向量场景**：ColBERT / SPLADE 类 late interaction 模型每文档生成数十-数百 token-level vector。Milvus 论文 μ ≤ 几十；token-level 场景规模大三个数量级，两个算法是否仍有效？wiki 未覆盖
- **聚合函数 g 的 design space**：weighted sum / median / min / max 各自最优算法可能不同；论文统一用 fusion / iterative，没分类讨论
- **跨 entity index 的 graph 类支持**：HNSW / NSG 是否能直接 multi-vector？论文 IVF_FLAT 才是默认，graph 类未实测
- **Embedding 模型联合训练对算法的影响**：当 v₀, v₁ 来自同一个多模态模型（CLIP）时，分布相关性可能让 vector fusion 工作更好；独立模型时则相反——assumption 论文未量化
- **多 vector 下的 GPU 优化**：[Faiss-GPU](../systems/faiss.md) IVFADC + [WarpSelect](../concepts/warpselect.md) 是单 vector 优化；fused vector 的 GEMM 形态变了，是否仍最优？
- **流式 multi-vector**：动态加入第 μ 个 vector 维度（schema 演化）——LSM segment 模型下如何重组？[per systems/milvus.md Open Q] 已 flag

## Cited Pages

- [systems/milvus.md](../systems/milvus.md)
- [systems/vbase.md](../systems/vbase.md)
- [concepts/product-quantization.md](../concepts/product-quantization.md)
- [concepts/relaxed-monotonicity.md](../concepts/relaxed-monotonicity.md)
- [topics/mips-vs-l2-nn.md](./mips-vs-l2-nn.md)
- [topics/topk-vs-iterator-model.md](./topk-vs-iterator-model.md)
