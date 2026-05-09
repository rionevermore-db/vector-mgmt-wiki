---
title: Block Shuffling（disk graph index 的 block-level 数据布局优化）
type: concept
sources: [wang-2024-starling]
related: [vamana.md, hnsw.md, nsg.md, product-quantization.md, ../systems/starling.md, ../systems/diskann.md, ../systems/spann.md, ../systems/milvus.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md]
created: 2026-05-09
updated: 2026-05-09
---

# Block Shuffling

**TL;DR**: Starling [wang-2024-starling §4.1] 提出的 disk graph index 数据布局问题——**给定 graph index G(V,E)，把 |V| vertex 分配到 ρ block 使 OR(G)（block 内邻居 overlap 比）最大化**。Starling 论文证明 **block shuffling 问题 NP-hard 且不存在多项式时间近似算法 with finite approximation factor unless P=NP**（Theorem 4.1）。三个启发式：BNP / BNF / BNS。这是 wiki 内首个把"disk graph index 数据布局"作为可量化优化问题形式化的工作——之前 [DiskANN](../systems/diskann.md) 默认 ID-consecutive 顶点同 block（OR(G) ≈ 0），94% 读到的 disk block 数据浪费；Starling-BNF 实测 OR(G) 0.34-0.87，5-7× 提升 vertex utilization。

## 提出背景

[wang-2024-starling §3.1, §4.1]

Disk-resident graph index（[DiskANN](../systems/diskann.md) / Disk-NSG / Disk-HNSW）在 segment-level（≤10GB disk）的 search latency 92.5% 用于 disk I/O。两个根本问题：

**Problem 1 (poor data locality)**：DiskANN 把 ID-consecutive vertex 分到同 block——但 ID-consecutive ≠ graph-adjacent。每个 4KB block 默认装 16 vertex（128-d uint8 + 32 邻居 ID）；search 时只用其中 1 个（target vertex），其他 15 个跟 query 无关——94% 读到的 disk block 浪费。

**Problem 2 (long search path)**：Random/fixed entry point 距离 query 邻域可能数百 hops；每 hop 一次 disk I/O。

Block shuffling 攻击 Problem 1——通过重排 vertex 到 block 让一个 block 内 vertex 互为 graph 邻居。

## 形式化定义

[wang-2024-starling §4.1 Def 1, Def 2, Eq 5]

**Block-Level Graph Layout**：scheme 把 |V| vertices 分配到 ρ blocks（ρ = ⌈|V|/ε⌉，ε = block 装下的 max vertex 数）。

**Overlap Ratio (vertex-level)**：

```
OR(u) = |B(u) ∩ N(u)| / (|B(u)| - 1)   if |B(u)| > 1, else 0
```

B(u) = 包含 u 的 block；N(u) = u 的图邻居集。直观：u 的 block 内**除 u 外有多少比例是 u 的邻居**。

**Overlap Ratio (graph-level)**：

```
OR(G) = (1/|V|) · Σ_{u∈V} OR(u)
```

OR(G) ∈ [0, 1]——1 表示完美 locality（每 block 内 vertex 互为邻居），0 表示完全无关。

**Block Shuffling 问题**：给定 graph layout，找新 layout 最大化 OR(G)。

## NP-hardness 证明

[wang-2024-starling Theorem 4.1 + Appendix A]

通过 reduction from **triple shuffling problem** [10, 59]（strongly NP-complete）：

> **Triple shuffling**：给定 t = 3·ρ 个整数 α₀, ..., α_{t-1}，threshold Ω 满足 Ω/4 < αᵢ < Ω/2 且 Σαᵢ = ρ·Ω；找方法把它们分组成 ρ triples，每 triple 之和 = Ω。

Reduction 构造：每 αᵢ 对应一个大小 αᵢ 的 clique；各 clique 联合成 graph G。证明 G 上 block shuffling 存在解 ↔ triple shuffling 有解。

**进一步**：不存在多项式时间近似算法 with finite approximation factor——若有，则可解 triple shuffling 的"是否最优"判定问题（NP-hard）→ 矛盾。

→ block shuffling **没有理论 best-effort 上界**——这正当化"多个 heuristic 算法各取所长"的工程路径。

## 三个启发式算法

[wang-2024-starling §4.1 Algorithm I/II/III]

### Algorithm I: BNP (Block Neighbor Padding)

**思路**：按 ID 顺序填 block——遇到 vertex u，尝试把它和它的邻居放进当前 block；block 满则开新 block。

**复杂度**：O(|V|)（一遍扫描）

**OR(G)**：低——只能利用 u 与已分配邻居的关系，不优化 u 已分配的邻居与 u 未分配的邻居。

**适用**：build 时间紧的场景；作为后续算法的 init layout。

### Algorithm II: BNF (Block Neighbor Frequency)——Starling 默认

**思路**：iterative。每轮：
1. 清空所有 block
2. 对每 vertex u，找包含**最多 u 邻居**的 block；若该 block 未满则放 u
3. 否则按邻居频率降序找下个未满 block；若全满则开新 block
4. β 轮迭代或 OR(G) 增益 < τ 时停

**复杂度**：O(β·o·|V|)（β = max iterations，o = average out-degree）

**OR(G)**：mid-high。比 BNP 更好——主动让 vertex 紧贴最大邻居 block。

**Starling 默认 β = 8, τ = 0.01**——实测 BIGANN OR(G) 0.34，时间为 disk graph 构造的 9.5%。

### Algorithm III: BNS (Block Neighbor Swap)

**思路**：从 BNP/BNF 输出 layout 开始，**iteratively swap 邻居跨 block**：对 u 的两个邻居 a, e（位于不同 block B(a), B(e)），找 B(a)、B(e) 中 OR 最低的 vertex 互换。

**复杂度**：O(β·o³·ε·|V|)（最高）

**OR(G)**：highest——三个算法中最优；NN-Descent 风格的局部优化。

**Lemma 4.2**：BNS 中 OR(G) 是 monotonically non-decreasing function of iterations β——保证不退化。

### 算法选择

[wang-2024-starling §4.1 末尾]

| 算法 | 速度 | OR(G) | 适用场景 |
|---|---|---|---|
| BNP | 快（一遍） | 低 | 紧迫 build 时间 |
| **BNF (default)** | mid | mid-high | **生产平衡** |
| BNS | 慢 | highest | offline pre-build OK，搜索性能极致 |

## 实测效果

[wang-2024-starling §6.5 + Fig 9]

OR(G) on BIGANN 33M / DEEP 11M / SSNPP 16M / Text2image 5M：

| Method | BIGANN | DEEP | SSNPP | Text2image |
|---|---|---|---|---|
| **DiskANN (no shuffle)** | **0.0625** | 0.1429 | 0.1111 | 0.2500 |
| Starling-BNP | mid | mid | mid | mid |
| **Starling-BNF (default)** | **0.3438** | **0.4429** | **0.4111** | **0.8760** |
| Starling-BNS | highest | highest | highest | highest |

→ DiskANN 上 OR(G) ≈ 0 表明 **94% block 数据浪费**；Starling-BNF 5-7× 提升 vertex utilization ratio (ξ)。

## 与图分区的关系

[wang-2024-starling §4.1 Remarks]

Block shuffling 类似但不同于经典 graph partitioning：

| 维度 | 社交网络（power-law）| Vector graph index |
|---|---|---|
| 度分布 | 长尾（hub vertices）| **uniform**（每 vertex 度数一致 ~32）|
| 邻居 cluster 性 | 强（power-law 自带） | **弱**（高维 vector 邻居 scatter across clusters，~50% long links） |
| 现成算法 | METIS / KGGGP / 大量 partition tool 适用 | **不适用**——KGGGP 比 BNF OR(G) 低 40% |

→ vector graph index 的邻居关系**与社交网络根本不同**，需要专门的 block shuffling 而非通用 graph partitioning。

## Time / Space Cost

[wang-2024-starling §4.1 Time/Space cost]

- **Time**: BNF 仅扫 vertex + 简单统计，**不做 vector 距离计算**——比 graph 构造快很多。BIGANN 实测 BNF 占总 index processing **~9.5%**
- **Space**: Disk graph **size 不变**（不增加邻居或顶点，只调换 vertex 顺序）

## 实现要点

[wang-2024-starling §4.1 + GitHub zilliztech/starling]

- 工作于已构造的 disk graph index 之上——**完全 orthogonal to graph algorithm choice**（[Vamana](./vamana.md) / [HNSW](./hnsw.md) / [NSG](./nsg.md) 都可）
- block size 不限于默认 4KB——可扩展到 8KB / 16KB（adjust block layout 维度）
- BNF 多线程并行（实测 64 thread on Threadripper）
- BNP / BNF / BNS 可串联（BNF init → BNS 精细化）

## 与 wiki 已 ingest concept 的关系

### 与 [Vamana](./vamana.md)

Vamana 是 [DiskANN](../systems/diskann.md) 的图算法核心。Block shuffling 不修改 Vamana 算法本身，只重排 vertex 到 block→ Vamana α-controlled RobustPrune 的 graph topology 不变。

### 与 [HNSW](./hnsw.md)

HNSW 多层 graph 中，**仅 layer-0**（contains all vectors）需要 block shuffling；upper layers 全在内存。Starling-HNSW 把 upper layers 当 in-memory navigation graph + layer-0 走 block shuffling。

### 与 [NSG](./nsg.md)

NSG 单层 graph，与 Vamana 同样可 block shuffle。Starling-NSG 实测 2× 快于 Disk-NSG baseline。

### 与 [Product Quantization](./product-quantization.md)

Block shuffling 与 PQ **正交**——Starling 同时用 block shuffling（locality）和 PQ（neighbor distance approximation）；两者攻击 disk graph 的不同 inefficiency。

## Open Questions

- **更高 OR(G) 的算法**：BNS 是否已接近 NP-hard 上界？wiki 未量化"BNS vs optimal"差距
- **动态 graph 下 block shuffling**：Starling 假设 static graph；增量插入时 OR(G) 如何漂移？BNF 重跑成本在动态场景如何摊销？论文 §7 提"周期 merge to disk"模式但未深入
- **Block shuffling + [LIRE](./lire.md) 增量再平衡**：理论上同 spirit（重排数据 layout）但作用层不同（LIRE 在 IVF cluster 边界 / Block shuffling 在 graph block 内部）；联合优化未探索
- **Block shuffling 在 [SPANN](../systems/spann.md) 的可行性**：SPANN 是 IVF + posting list（不是 graph index）；block shuffling 对 SPANN 不直接适用——但 posting-list-level 的 locality 优化是类似问题
- **Block shuffling 与 [RaBitQ](./rabitq.md) 集成**：RaBitQ 替代 PQ short codes 后，每 vertex storage ~1/4——更多 vertex per block → 改变 ε → 改变 OR(G) 上限。重新优化 layout 可能进一步提升
- **更大 block size 的回报**：Starling 默认 4KB；8KB/16KB 时 OR(G) 是否更高？Trade-off：单次 disk read 数据量 ↑ 但相对 useful data 比例可能 ↑
- **GPU disk graph index 的 block shuffling**：Starling §8 提 future work；GPU 内存层级与 disk block 不同概念，block shuffling 在 GPU 形态需重新形式化

Cited by: 待 query 引用
