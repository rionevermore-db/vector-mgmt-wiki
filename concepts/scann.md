---
title: ScaNN（Anisotropic Vector Quantization）
type: concept
sources: [guo-2019-scann]
related: [product-quantization.md, hnsw.md, nsg.md, ../topics/mips-vs-l2-nn.md, ../benchmarks/scann-glove1.2m-mips.md]
created: 2026-05-07
updated: 2026-05-07
---

# ScaNN

**TL;DR**: 把传统 PQ 优化的 reconstruction error `||x − x̃||²` 换成 **score-aware loss** —— 平行于 x 的残差分量被加重惩罚（h_∥ ≥ h_⊥）。在 MIPS 任务下 Glove1.2M 上 200 bit code Recall1@10 从 0.83 提升到 0.91。Google 开源的 ScaNN 库即基于此。[guo-2019-scann §3-5]

## 提出背景

Ruiqi Guo, Philip Sun, Erik Lindgren, Quan Geng, David Simcha, Felix Chern, Sanjiv Kumar（Google Research），ICML 2020（arXiv 2019）。

针对的核心矛盾：传统 [PQ](./product-quantization.md) 及其全部后继（OPQ、LSQ、QUIPS）都最小化 reconstruction error，但**这是 L2-NN 的目标，不是 MIPS 的目标**。MIPS 任务下不同 (q, x) 对的重要性差异巨大：高 `<q,x>` 对更可能是 top-k，对它们的量化误差应被加重。详见 [MIPS vs L2-NN](../topics/mips-vs-l2-nn.md)。

## 理论核心：误差的各向异性分解

### Score-aware loss

对任意 score 加权函数 w：

`ℓ(x, x̃, w) = E_q~Q [w(<q,x>) · <q, x − x̃>²]`

- q 假设在单位球上均匀分布（isotropic query distribution）
- w 是 monotonically non-decreasing 的非负函数

[guo-2019-scann Definition 3.1]

### 关键定理（Theorem 3.2）

ℓ 总能分解为：

`ℓ = h_∥(w, ||x||) · ||r_∥||²  +  h_⊥(w, ||x||) · ||r_⊥||²`

其中：
- `r_∥(x, x̃) = <(x − x̃), x> · x / ||x||²` —— 残差**平行于 x** 的分量
- `r_⊥(x, x̃) = (x − x̃) − r_∥` —— 残差**正交于 x** 的分量

### 关键不等式（Theorem 3.3）

**对任意 monotonically non-decreasing w（t≥0），h_∥ ≥ h_⊥**。

直觉：MIPS 排序对"沿 x 方向的偏差"更敏感（直接改变 `<q,x>`），对正交偏差不敏感（投影掉了）。所以最优 codebook 应优先压缩平行误差。

### 实用 w：阈值指示函数

取 `w(t) = I(t ≥ T)`：

| T | η = h_∥/h_⊥ | 退化情况 |
|---|---|---|
| 0 | 1 | 标准 PQ |
| **0.2** | **≈ 4.125** | 推荐配置（高维极限，Theorem 3.4） |
| → ‖x‖ | ∞ | 完全只惩罚平行误差 |

T 是论文唯一引入的超参数。Glove1.2M 上 T=0.2 接近最优（Fig 3a）。

## 算法（Anisotropic Product Quantization, §4）

迭代式 codebook 学习，与标准 PQ 同框架（Lloyd-style）：

1. **Initialization**：随机选 k 个 codeword
2. **Partition Assignment**：每个 x 分到使 `ℓ(x, c, w)` 最小的 codeword
3. **Codebook Update**：闭式解（Theorem 4.2）：
   `c_j* = (Σ h_⊥·I + Σ (h_∥−h_⊥)/||x||² · x·x^T)^(−1) · Σ h_⊥·x`
4. 循环 2–3 直到收敛或达到最大迭代

**当 h_∥ = h_⊥ 时退化为 k-means 平均更新** —— 即标准 [PQ](./product-quantization.md)。这一兼容性意味着 ScaNN 几乎是"PQ 的一行 loss 替换"。

## 与传统 PQ 的对比

| | 传统 [PQ](./product-quantization.md) | ScaNN (Anisotropic) |
|---|---|---|
| Loss | `||x − x̃||²`（reconstruction） | `h_∥·||r_∥||² + h_⊥·||r_⊥||²`（score-aware） |
| 假设的查询分布 | 隐式：所有 (q, x) 等权 | 显式：q 均匀；高 `<q,x>` 对加权 |
| 主要适用任务 | L2-NN、ANN（一般） | **MIPS 优先** |
| Codebook update | k-means 平均 | 加权伪逆（闭式） |
| 超参数增量 | 0 | 1 个（T） |
| 工程改造成本 | — | 仅替换 loss 与 update 公式 |

## 经验数字（Glove1.2M）

| Setting | Recall1@10 |
|---|---|
| 200 bit, 传统 reconstruction | 0.83 |
| 200 bit, score-aware (T=0.2) | **0.91** |

[guo-2019-scann Fig 3a]

详见 [ScaNN on Glove1.2M MIPS](../benchmarks/scann-glove1.2m-mips.md)。

## 与 graph methods 的关系

在 MIPS 任务的 high-recall 区间（≥ 0.95），ScaNN 单线程 QPS 在 Glove1.2M 上击败 [HNSW](./hnsw.md)（nmslib 与 faiss 实现）、NGT、SW-graph 等所有 ann-benchmarks 主流算法。[guo-2019-scann §5.3 + Fig 4b]

但**仅限 MIPS**。L2-NN 任务下 graph 路径仍然占优。详见 [topics/mips-vs-l2-nn.md](../topics/mips-vs-l2-nn.md)。

## 典型实现

- 作者实现：[`google-research/scann`](https://github.com/google-research/google-research/tree/master/scann)
- 工程栈：anisotropic PQ + SIMD-based ADC（Guo 2016b）+ 上层 vector quantization tree（Wu et al. 2017）做 IVF
- Bazel 构建，Python 接口；TensorFlow Serving 集成

## Open Questions

- **q 的分布假设**：论文要求 q 在单位球上均匀。真实推荐场景下 query 嵌入分布往往有结构（hot queries、用户聚类）。这个假设的鲁棒性未充分讨论。
- **低维下的 η 计算**：Theorem 3.4 的极限解析式只在 d→∞ 时成立。低维（d<100）需要数值积分 h_∥ / h_⊥，论文未明示工程细节。
- **多目标任务**：ScaNN 优化 MIPS。如果应用同时需要 L2-NN（比如混合检索）怎么办？切换 codebook 还是另建索引？
- **与 graph 方法融合**：ScaNN 自家也用了 vector quantization tree（IVF 类）做粗量化，但未尝试 [HNSW](./hnsw.md)-as-coarse-quantizer。HNSW + 各向异性 PQ 的混合栈是开放方向。
