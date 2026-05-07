---
title: DiskANN（System）
type: system
sources: [subramanya-2019-diskann, chen-2021-spann]
related: [../concepts/vamana.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, faiss.md, spann.md, milvus.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/attribute-filtering.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/spann-vs-diskann-billion.md]
created: 2026-05-07
updated: 2026-05-07
---

# DiskANN

**TL;DR**: Microsoft Research 开源的 SSD-resident ANN 系统，基于 [Vamana](../concepts/vamana.md) graph 算法。把 PQ 压缩向量放 DRAM、Vamana graph + 全精度向量放 SSD，**用图遍历做导航 + SSD 全精度 re-rank**。在单机 64 GB RAM 上跑 1B SIFT，**1-recall@1 = 98.68% @ <5 ms latency**，>5000 QPS。是 [Faiss](./faiss.md) 在 SSD 维度的主要竞品。[subramanya-2019-diskann §3-4]

## 架构图

```
┌──────────────────────────────┐
│  DRAM (64 GB)                │
│  ├─ PQ-compressed vectors    │  ← 32 byte/vector，导航用
│  ├─ Vertex cache (~3-4 hops) │  ← 高频节点
│  └─ Beam search state        │
├──────────────────────────────┤
│  SSD (~hundreds of GB)       │
│  ├─ Vamana graph edges       │  ← 邻居 ID 列表
│  └─ Full-precision vectors   │  ← 同扇区 piggyback
└──────────────────────────────┘
```

[subramanya-2019-diskann §3.1-3.2]

## 数据流 / 控制流

### 索引构建

1. 数据切到 k=40 个 k-means 簇（每点分配到 ℓ=2 最近簇 → overlapping）
2. 每个 shard（约 N·ℓ/k 点）独立构建 [Vamana](../concepts/vamana.md) 索引（in-memory）
3. 合并所有 shard 的图：取边集的简单 union
4. 同时 train Product Quantizer，把 PQ codes（32 byte/vector）存内存
5. 全精度向量按图节点 ID 顺序写到 SSD（每个节点一个 4 KB 扇区，含 R 邻居 ID + 全精度坐标）

详见 [subramanya-2019-diskann §3.1]。

### 查询路径（Beam Search）

[subramanya-2019-diskann §3.3]

```
input: query xq, beam width W (典型 4-8)
state: candidate list L, visited set V

while L 还有未访问点:
  从 L 取 W 个最近的未访问候选 → 一次性从 SSD 读它们的扇区
                                    （每扇区含全精度坐标 + R 邻居 ID）
  用 DRAM 中的 PQ codes 估算到这些邻居的近似距离 → 加入 L
  顺手用刚读的全精度坐标精化 L 中的距离
return top-k from L (using full-precision distances)
```

关键：**beam width W 是一次磁盘 I/O 批量读邻居数**。

- W=1：退化为 GreedySearch（多 round-trips）
- W=8：4-8× 减少 round-trips
- W>16：单次读太多扇区，浪费 SSD 带宽

## 关键设计决策

### 1. 全精度 re-ranking 是免费的（§3.5）

SSD 4 KB 扇区可以同时装下 R=128 个邻居 ID（128·4 = 512 B）+ 全精度向量（如 128-d × 4 B = 512 B）。**读邻居顺手就拿到全精度坐标**，用来对路径上访问的所有点做精确 re-rank，无需额外 SSD 访问。

**Trade-off**：用 PQ 压缩做导航（容忍精度损失换 RAM 容量）+ SSD 取出全精度做 ranking（避免 PQ 的最终精度损失）。这是 [PQ](../concepts/product-quantization.md) / [Faiss IVFPQ](./faiss.md) 做不到的。

### 2. Beam Search 替代 GreedySearch（§3.3）

每次从 SSD 批量取 W 个邻居而非一个。**Trade-off**：W 大 → 减少 round-trips 但增加 SSD bandwidth 浪费；W 小 → 多 round-trips 但每次读得精确。

### 3. 顶级节点缓存（§3.4）

离 medoid C=3-4 跳的所有节点缓存到 DRAM。**Trade-off**：占 DRAM（节点数随 C 指数增长）vs 减少 SSD 访问。

### 4. Merged Build（§3.1）

单 shot Vamana 构建 1B SIFT 需 1100 GB RAM；merged 版用 k=40 / ℓ=2 overlapping shard，每 shard 64 GB RAM 内构建，最后简单合并图边。**Trade-off**：构建时间 5 天 vs 单 shot 2 天，但内存峰值 64 GB vs 1100 GB。

### 5. PQ 仅用于导航不用于 ranking

PQ 失真大 → 图遍历可能走偏路径，需要更多 hops；但终点全精度 re-rank 保证 recall 不被 PQ 限制。**Trade-off**：放弃 [PQ](../concepts/product-quantization.md) 的 final ADC 距离，换更高 recall 上限。

## Scale 边界

- SIFT1B（128-d, 1B 点）：64 GB RAM 工作站可行，348 GB 索引 on SSD，5 天构建（merged）
- 论文未测 10B+；SSD 容量是首要瓶颈
- 论文未测 trillion-scale —— 需要分布式 SSD（[Faiss trillion-scale 案例](../benchmarks/faiss-trillion-scale.md) 走的是不同路线）

## 与 Faiss 的对比

| | DiskANN | [Faiss](./faiss.md) |
|---|---|---|
| 主要存储介质 | **SSD** + DRAM (PQ codes) | **DRAM**（CPU）/ HBM（GPU） |
| 算法 | [Vamana](../concepts/vamana.md) graph | IVFPQ / HNSW / NSG / 多种 |
| 1B 单机方案 | 64 GB RAM + SSD | 必须 PQ 压缩到全内存 |
| 1B Recall @ <5ms | **98.68%** | IVFOADC+G+P-32 plateau 62.74% |
| 内存预算 | 32 byte/vec PQ in DRAM | 通常更大 |
| 增量 add | 不支持 | 部分支持 |
| 主语言 | C++ | C++ |
| GPU | 不支持 | 支持 |

[subramanya-2019-diskann §4.4 + Fig 2a]；[douze-2024-faiss-library §2 + §5.5 Fig 8] 也对比 Faiss vs Vamana on Deep10M。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

## 与 SPANN 的对比

[SPANN](./spann.md) 是 Microsoft **同期、同目标、对立路线**的对手系统（NeurIPS 2021，DiskANN 是 NeurIPS 2019）：

| | DiskANN | [SPANN](./spann.md) |
|---|---|---|
| 算法路线 | **Graph (Vamana) + SSD** | **Inverted file + SSD** |
| 是否用 PQ | 是（导航） | 否（全精度） |
| 90% recall 延迟 | ~3-4 ms | **~1 ms** |
| 内存预算 | 32 byte/vec PQ | ~16% N × centroid 大小 |
| 公平对比下 | 多 latency budget 下混合胜率 | **SPANN 在低 latency budget 下系统性领先** |

[chen-2021-spann §4.2 + Fig 6]；详见 [SPANN vs DiskANN benchmark](../benchmarks/spann-vs-diskann-billion.md)。

**SPANN 的核心优势**：inverted file 的 SSD 访问模式（少量大读）天然比 graph 的 SSD 访问模式（多次小读）友好；不需要 PQ 即可控制 IO。

## 生产案例

- **Microsoft 内部**：Bing search、Microsoft 365 嵌入检索（Microsoft Research 输出）
- **后继 Filtered-DiskANN**：[douze-2024-faiss-library §2] 提到的过滤检索扩展，支持元数据查询
- **OOD-DiskANN**：处理 out-of-distribution queries（同上）
- **FreshDiskANN**：支持 streaming updates（同上）

## Open Questions

- **Merged 方案的 connectivity 是经验性的**：overlapping clusters 提供"足够"连通是观察不是证明
- **PQ 失真的 cascading effect**：PQ 越粗糙，图遍历越绕路，越多 SSD 读；论文未量化此 trade-off
- **SSD 寿命**：>5000 QPS × 几百 μs 随机读 = 高 IOPS 持续负载；commodity SSD 寿命影响未讨论
- **网络存储情况下不可用**：所有 latency 假设基于本地 NVMe SSD；分布式存储 / 远程块设备完全失效
- **更新与删除**：与 [NSG](../concepts/nsg.md) 一样不支持增量；merged 方案下加点更难
- **和 Faiss 的最优组合**：理论上 [Faiss](./faiss.md) 的 IVF + Vamana coarse quantizer 是新点子，但当前两个生态独立

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md)
