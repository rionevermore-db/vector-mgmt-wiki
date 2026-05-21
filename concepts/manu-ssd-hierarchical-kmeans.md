---
title: Manu SSD-Aware Indexing（NeurIPS 2021 BigANN Winner）
type: concept
sources: [guo-2022-manu]
related: [../systems/milvus.md, ../systems/spann.md, ../systems/diskann.md, ../topics/disk-vs-memory-ann.md]
created: 2026-05-08
updated: 2026-05-08
---

# Manu SSD-Aware Indexing

**TL;DR**: Manu 论文 §4.4 描述的 SSD-resident 向量索引——**hierarchical k-means** 把向量分到 4KB SSD-aligned 块 + **多次 LSH-style 复制** 提升 boundary recall + DRAM 中保留所有 cluster centers。**赢得 NeurIPS 2021 BigANN challenge Track 2 (search with SSD)，比 baseline recall 高 60% 同 QPS**。是 [SPANN](../systems/spann.md) / [DiskANN](../systems/diskann.md) 之外的第三种 cluster-based SSD 路线。[guo-2022-manu §4.4]

## 提出背景

[guo-2022-manu §4.4]

SSD 经济学（论文给）：
- SSD 比 DRAM 便宜 ~100×
- SSD bandwidth 比 HDD 大 ~10×
- 但 SSD bandwidth 仍远小于 DRAM——需要谨慎 layout 与 index 结构

设计目标：让 query node 在**有限 DRAM** 下处理 billion-scale 向量索引——通过把多数数据下放 SSD，DRAM 只放索引 metadata + cluster centers。

## 三层结构

[guo-2022-manu §4.4]

### Layer 1：4KB SSD-aligned blocks

```
SSD 上每 4KB block 存一个 cluster:
   ┌───────────────────────────────┐
   │ Block (4 KB SSD-aligned)      │
   │ ├─ vec₁ (compressed via SQ)   │
   │ ├─ vec₂                       │
   │ ├─ ...                        │
   │ └─ vec_n（n 取决于 dim）        │
   └───────────────────────────────┘
```

- 论文论证："reading less than 4KB has the same cost as reading 4KB"——SSD I/O 单元
- bucket 大小若 ≈ 4KB（向量小时）则 1 bucket = 1 block；向量大时设 bucket = 4-8 倍 4KB

### Layer 2：DRAM 中的 cluster centers + index

```
DRAM:
   ┌──────────────────────────────────────┐
   │ All cluster centers (hierarchical)   │
   │   k-means 树结构                       │
   │   每 center 指向 SSD 上对应 block      │
   └──────────────────────────────────────┘
   ┌──────────────────────────────────────┐
   │ Vector search index over centers     │
   │   IVF-FLAT / HNSW etc                 │
   └──────────────────────────────────────┘
```

- DRAM 仅存 centers（数量 << 总向量数）
- 用 IVF-FLAT 或 HNSW 索引这些 centers
- query 时先在 DRAM 找最近 centers → 加载对应 SSD blocks → 扫描

### Layer 3：LSH-style 复制（关键创新）

[guo-2022-manu §4.4 末段]

**问题**：k-means 可能把"接近 query 但跨 cluster boundary"的向量分到错误 bucket → recall 低。

**Manu 解法**（**论文核心创新**）：
- 用 **LSH 思路** [Datar et al. 2004]：构造**多个独立的 hierarchical k-means 树**
- 同一向量在每棵树都被分配到一个 bucket → **一个向量在 SSD 中复制多次**（典型 4-8 次）
- query 时**所有树的 cluster centers 都在 DRAM 检索** → 命中多 candidates → 合并

**Trade-off**：
- ✓ recall 显著提升（NeurIPS 2021 实测 60% over baseline at same QPS）
- ✗ SSD 占用 4-8×（向量复制 4-8 份）
- ✗ DRAM cluster centers 4-8× 但仍很小

## 与 [SPANN](../systems/spann.md) closure clustering 的对比

[per systems/spann.md "Closure Clustering 解决 boundary issue" + guo-2022-manu §4.4]

两个方案解决**同一问题**（cluster-based SSD ANN 的 boundary recall）：

| | SPANN closure clustering | **Manu hierarchical k-means + LSH** |
|---|---|---|
| 思路 | 边界向量复制到多个最近簇 | **多次独立 hierarchical k-means** |
| 复制粒度 | 选择性（仅边界向量） | **全数据集复制 4-8 次** |
| 复制 selectivity 控制 | RNG rule（避免冗余复制） | LSH-style（独立 hash） |
| 复制系数 | 最多 8 replicas（边界） | 4-8× 全数据集 |
| 算法新颖性 | RNG-based pruning | **LSH-style multi-tree** |
| 实测 | Bing 千亿生产 | NeurIPS 2021 winner（60% recall ↑） |
| 集成系统 | SPTAG / [SPANN](../systems/spann.md) | [Manu / Milvus 2.x](../systems/milvus.md) |

→ 两者**同代浪潮**（2021 NeurIPS / SOSP / VLDB），不同思路。SPANN 选择性复制更省 SSD，Manu 全复制更简单 + 更高 recall。

## 与 [DiskANN](../systems/diskann.md) Vamana + SSD 的对比

| | DiskANN | Manu SSD index |
|---|---|---|
| 算法路线 | Graph (Vamana) + PQ DRAM 导航 + SSD 全精度 re-rank | **Cluster-based + DRAM centers + SSD scan** |
| Boundary 处理 | α-controlled graph 边长 | LSH-style 多次 k-means |
| Per-query SSD I/O | 多次小读（每跳一次） | 少量大读（K 个 4KB block） |
| Quantization | PQ codes in DRAM | **SQ scalar quantization in SSD blocks** |
| Recall 上限 | 98.68%（SSD 全精度 re-rank） | docs 仅给 NeurIPS 60% improvement，绝对值未明示 |

→ Manu 路线**更类 [SPANN](../systems/spann.md)**（cluster-based + posting list on SSD），不像 DiskANN（graph + SSD）。

## 关键性能数据

[guo-2022-manu §4.4]

**NeurIPS 2021 BigANN Track 2 (search with SSD) winner**:
- 比 baseline at same QPS 提升 **recall 60%**
- 该 challenge 的 baseline 是 DiskANN
- 论文未直接给 absolute QPS / recall 数字（参考 challenge 网站）

实验注：[Sun et al. 2022 NeurIPS 2021 winner challenge results paper] 给具体数字，该 results paper 本身 wiki 未 ingest;但**竞赛结构 + Track 2(SSD)冠军 BBANN(= Zilliz,本方案)已由 [benchmarks/big-ann-benchmarks.md](../benchmarks/big-ann-benchmarks.md) 记录**(2026-05-21 lint 部分闭合)。

## Open Questions

- **LSH-style multi-tree 的最优复制次数**：论文 4-8× 是经验值；不同数据分布最优值未量化
- **混合 quantization**：当前只用 SQ；是否能与 OPQ / [ScaNN](./scann.md) anisotropic loss 结合？
- **Update 在该 SSD index 上的成本**：vector update 需更新 4-8 个 replicas + 重建 cluster center（如 cluster 漂移）。Manu 论文未深入；可能因此论文用 stream indexing + 周期 rebuild 代替 in-place（与 [SPFresh](../systems/spfresh.md) 思路相反）
- **vs SPANN closure clustering 直接对比**：两个 cluster-based SSD 方案应直接对比，但论文未做；[benchmarks/spann-vs-diskann-billion.md] 也未含 Manu
- **[Milvus](../systems/milvus.md) 2.x 中是否当前默认使用此索引**：论文写于 2022；2.6.x docs 列 DISKANN 而非这个 hierarchical k-means + LSH —— 可能已被 DiskANN 集成取代
- **NeurIPS 2021 winner 论文细节**：Sun et al. 2022 challenge results paper（[guo-2022-manu ref 72]）有更深技术;该 results paper wiki 未 ingest（但竞赛 + BBANN 冠军已由 [benchmarks/big-ann-benchmarks.md](../benchmarks/big-ann-benchmarks.md) 覆盖）

Cited by: 待 query 引用
