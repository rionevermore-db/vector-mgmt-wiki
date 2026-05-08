---
title: SPANN（System）
type: system
sources: [chen-2021-spann, gao-2024-rabitq]
related: [../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/lire.md, ../concepts/relaxed-monotonicity.md, ../concepts/rabitq.md, diskann.md, faiss.md, milvus.md, spfresh.md, pinecone.md, vbase.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/topk-vs-iterator-model.md, ../benchmarks/spann-vs-diskann-billion.md, ../benchmarks/vbase-8queries-recipe1m.md, ../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]
created: 2026-05-07
updated: 2026-05-08 (RaBitQ)
---

# SPANN

**TL;DR**: Microsoft 开源的 SSD-resident ANN 系统，走 inverted file 路线。**Centroids 在内存（占 ~10% 总向量）+ posting list 全精度在 SSD**。在三个 billion-scale 数据集上比 [DiskANN](./diskann.md) 在 90% recall 时快 **2×**，已部署到 Microsoft Bing 几千亿规模。是 DiskANN 的同期对手（同时期、同一公司、对立算法路线）。[chen-2021-spann §3-4]

## 架构图

```
┌──────────────────────────────────────────┐
│  DRAM (~32 GB for 1B SIFT)               │
│  ├─ SPTAG 内存索引（树 + RNG graph）       │  ← 中心点导航
│  ├─ Posting list centroids (~16% N)      │  ← 簇中心点
│  └─ Closure / pruning 控制状态            │
├──────────────────────────────────────────┤
│  SSD (~hundreds of GB)                   │
│  └─ Posting lists（全精度，full vectors）  │  ← 12 KB / 48 KB per list
└──────────────────────────────────────────┘
```

[chen-2021-spann §3]

## 数据流 / 控制流

### 索引构建

1. **Hierarchical Balanced Clustering (HBC)**：迭代用 k-means 把数据切到小簇，直到每个 posting list 字节数 ≤ 上限（12 KB byte vector / 48 KB float vector）
2. 中心点用簇内最接近 centroid 的实际向量代替（减少导航的"假"中心）
3. **Closure clustering**：边界向量复制到多个最近簇（最多 8 replicas），用 RNG rule 避免相似簇间冗余复制
4. 内存中建 SPTAG（Microsoft 自家 ANN 库 [12 in chen-2021-spann]）over centroids，亚毫秒级 nearest-centroids 查询

### 查询路径

```
input: query q, K (number of posting lists to search)

1. SPTAG 内存索引中找 q 最近的 K 个 centroids → c_i1, ..., c_iK
2. Query-aware dynamic pruning：仅保留满足
   Dist(q, c_ij) ≤ (1+ε₂) × Dist(q, c_i1)
   的 c_ij（ε₂=6.0 for recall@1）
3. 从 SSD 顺序读对应 posting list，全精度计算距离
4. 维护 top-k 候选，返回最优
```

[chen-2021-spann §3.2.3, Eq 3]

## 关键设计决策

### 1. 不用 PQ —— 全程全精度（§3）

与 [Faiss IVFPQ](./faiss.md) / [DiskANN](./diskann.md) 都把 PQ 编码放内存不同，SPANN **完全不用量化**：内存只放 centroids（小），SSD 放完整全精度向量。

**Trade-off**：

- 优势：避免 PQ 失真天花板（recall 上限不被量化限制）
- 优势：simpler，不需要 train PQ codebook
- 代价：SSD 上的 posting list 比 PQ codes 大几倍（128-d × 4 byte vs 32 byte PQ code）

### 2. Closure Clustering 解决 boundary issue（§3.2.2）

朴素 IVF 把每个向量分到唯一最近簇 → query 真实最近邻可能落在 query 没访问的簇里。SPANN 用 closure：

`x ∈ X_ij ⟺ Dist(x, c_ij) ≤ (1+ε₁) × Dist(x, c_i1)` [Eq 2]

边界向量复制到多个簇。**Trade-off**：posting list 增大约 8× → SSD 占用增加，但 recall 显著提升。

### 3. RNG Rule 避免 closure 冗余（§3.2.2 + Fig 5）

如果两个 closure 簇互相很近，复制相同向量到两边是浪费。RNG 规则：跳过 cluster ij 当 `Dist(c_ij, x) > Dist(c_ij, c_{j-1})`。**保留向量在不同方向上的代表簇**而不是同一方向多份。

### 4. Query-aware Dynamic Pruning（§3.2.3）

不同 query 难度差异巨大（Fig 2：80% query 只搜 6 个 list 即可，难 query 需 114 个）。SPANN 不用固定 K，而是**动态决定**：基于 query 与最近 centroid 的距离阈值。

**Trade-off**：参数 ε₂ 需调；过小 → recall 损失，过大 → 失去 pruning 意义。

### 5. SPTAG 内存索引做 centroids 查询（§3.2.1）

中心数 ~16% 总向量数 = SIFT1B 的 1.6 亿 centroids。直接 brute force 查太慢。SPANN 调用自家 [SPTAG](https://github.com/microsoft/SPTAG) 库（树 + RNG graph 的 ANN 索引）做 centroid lookup，亚毫秒级。

**Trade-off**：增加一层"索引中的索引"复杂度，但避免 centroid 查询成为瓶颈。

## Scale 边界

- **1B SIFT / SPACEV1B / DEEP1B**：单机 32 GB RAM + SSD 工作站可行
- **Bing 几千亿规模**：分布式部署，详见 [chen-2021-spann §4.3]
- 论文未测 1T+；理论上每加 10× 数据需要 10× 中心数 + 10× SSD

## 与 DiskANN 的对比（核心差异）

| | SPANN | [DiskANN](./diskann.md) |
|---|---|---|
| 算法路线 | **Inverted file** | **Graph ([Vamana](../concepts/vamana.md))** |
| 内存放什么 | Centroids + SPTAG 索引 | PQ 压缩向量（32 byte/vec） |
| 是否量化 | **否**（全精度） | 是（PQ） |
| 单查询 SSD 访问数 | **少且批量**（~K 个 posting list） | 多次（图遍历每跳一次） |
| 90% recall 时延迟 | **~1 ms** | ~3-4 ms |
| Recall 上限 | 接近 100%（无量化失真） | 接近 100%（全精度 re-rank） |
| 算法新颖性 | 工程组合（HBC + closure + pruning） | Vamana 本身是新算法 |
| 适合 query 难度 | **均匀**（dynamic pruning） | 均匀（图本身适应） |

[chen-2021-spann Fig 6]；详见 [SPANN vs DiskANN benchmark](../benchmarks/spann-vs-diskann-billion.md) 与 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)。

## 与 Faiss 的对比

[Faiss IVFPQ](./faiss.md) 是 SPANN 的"全内存对应物"：

- Faiss IVFPQ：IVF + PQ，**全在内存**
- SPANN：IVF + 全精度，**centroids 在内存 / posting list 在 SSD**

SPANN 可看作"Faiss IVF 的 SSD 化版本"，但把"内存里 PQ codes"的预算换成"内存 centroids + SSD 全精度"。两者的 IVF 思想是同一个，区别在于把哪部分下放到磁盘。

## 分布式扩展（§4.3）

天然 multi-machine 友好：

- 用 multi-constraint balanced clustering + closure 把数据切到 M 台机器
- 用 **bin-packing** 把小分区合并打包到 M bin（同时平衡数据量 + 历史 query 访问频率）
- 32 partition 场景：每查询只 dispatch **~6.3 台机器**（vs random partition 的 32 台）
- **省 80.3% 的 IO / 计算成本**

[chen-2021-spann §4.3 + Fig 13-14]

## 生产案例

- **Microsoft Bing**：论文 §1 末尾明示"deployed into Microsoft Bing to support hundreds of billions scale vector search"
- **SPTAG 库**：[microsoft/SPTAG](https://github.com/microsoft/SPTAG) C++ 实现（与 SPANN 同库，SPANN 是其上层使用模式）

## Open Questions

- ~~**数据漂移下的退化**：closure clustering 训练好后簇分配冻结；新加点需要重 cluster~~ —— **2026-05-07 ingest [SPFresh](./spfresh.md) 已解**：同团队（Microsoft Research Asia）SOSP 2023 论文在 SPANN 之上加 [LIRE](../concepts/lire.md) 协议，实现 in-place 增量再平衡，0.4% 的插入触发本地 split + reassign，完全不需要全局 rebuild。详见 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)
- **小 latency budget 下的边界**：<1 ms 内 SPANN 是否还能保 recall？论文 Fig 6 边界不清晰
- **现代高维 embedding（768-d / 1024-d）**：posting list 上限对 byte 向量是 12 KB、float 向量是 48 KB（论文 §3.1）。128-d byte → 12 KB / 128 ≈ 96 vec；768-d float → 48 KB / 3072 ≈ 16 vec；1024-d float → 48 KB / 4096 ≈ 12 vec。绝对数字仍然急剧收缩，足以让 closure replicas 与 query-aware pruning 的成本结构改变 —— 论文未在此区间评估
- **Closure replicas = 8 是经验值**：与 DiskANN 的 W=4-8 一样是工程调参，无理论指导
- **HBC 树深度的尾部分布**：簇大小不均匀的极端情况未量化分析
- **VBASE+SPANN 集成实证**：[per zhang-2023-vbase §5.4 + benchmarks/vbase-8queries-recipe1m.md Table 8] VBASE 在 Azure Standard_L16s_v3 NVMe 上集成 SPANN，全部 8 query 类型可行；Q1 9.4 ms / 11.6 ms 99p, recall 0.9911；Q5 99p 519.7 ms（SSD 随机 IO 放大）。证明 SPANN 满足 [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md)——partition-based + SSD 索引可以走 VBASE iterator 范式。这是 wiki 内首次"in-memory graph (HNSW) + on-disk partition (SPANN)"用同一 query engine 的实证
- **SPANN posting list 引入 [RaBitQ](../concepts/rabitq.md)**：[chen-2021] §3 SPANN 论文明确反对量化（"避免 PQ 失真天花板"）；但 [gao-2024-rabitq] 提供的 unbiased + sharp error bound 量化器可能改变这个 trade-off——理论上 SPANN posting list 用 RaBitQ 编码后 (a) SSD 占用从 32D bits → D bits（4× 节省），(b) error bound 仍允许 100% recall （rerank 全精度从 SSD 读）。开放问题：SPANN 的 closure clustering 与 RaBitQ 的 normalization 假设是否兼容？(SPANN closure 把边界向量复制到多 cluster；RaBitQ normalize 基于 cluster centroid。复制边界向量 → 不同 normalize 基准 → 同 vector 多 quantization codes，是否影响 unbiasedness？理论分析未做)
