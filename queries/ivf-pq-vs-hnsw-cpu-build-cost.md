---
title: IVF-PQ vs HNSW 在 CPU 上的 build 成本——100M / nlist=10000 / k-means 训练 7+ 小时是否异常？
type: query
sources: [malkov-2016-hnsw, jegou-2011-pq, johnson-2017-faiss-gpu]
related: [../benchmarks/hnsw-vs-faiss-200m-sift.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../topics/gpu-vs-cpu-ann.md, ../topics/index-selection.md]
created: 2026-05-20
updated: 2026-05-20
---

# IVF-PQ vs HNSW 在 CPU 上的 build 成本

**Date**: 2026-05-20

**Question**:
在 CPU 上构建 IVF-PQ（nlist=10000）和 HNSW 各要多久？具体场景：100M 向量、CPU only。
实测发现 HNSW 20 分钟构完，但 IVF-PQ 跑了 7+ 小时（已知 k-means 训练就花了 7 小时）还没结束。这正常吗？

## TL;DR

**量级正常，不算严重异常。** 参考 [malkov-2016-hnsw §5.4 Table 3] @ 200M SIFT (128-d, 4×Xeon E5-4650 v2 CPU) 实测：HNSW efC=40 = **42 min**, Faiss IVFADC (OPQ + IMI + PQ) = **11-12 h**。100M (= 一半数据) 时 IVF-PQ build 预期 **5-7 h** 落在量级内，HNSW 20 min 也对得上比例。但 **wiki 没有** d=768、nlist=10000、ARM CPU 这套配置的 anchored 数据，具体调优空间需自测。

## Answer

### 1. Anchored baseline（200M SIFT, CPU）

[per benchmarks/hnsw-vs-faiss-200m-sift.md Table 3, malkov-2016-hnsw §5.4]

| 方案 | Build time | Peak memory |
|---|---|---|
| HNSW 1 (M=16, efC=500) | **5.6 h** | 64 GB |
| HNSW 2 (M=16, efC=40) | **42 min** | 64 GB |
| Faiss OPQ64,IMI2x14,PQ64 | **12 h** | 30 GB |
| Faiss OPQ32,IMI2x14,PQ32 | **11 h** | 23.5 GB |

硬件：4×Xeon E5-4650 v2 (32 核 Ivy Bridge-EP, 128 GB RAM, OpenBLAS)。IMI 2×14 是 inverted multi-index = 4096 × 4096 = 16M effective cells（比 nlist=10000 还细几个数量级），其 build 主体仍是 k-means + PQ codebook 训练。

**问题中的 100M ≈ 一半数据规模 → Faiss IVFADC 预期 5-7 h，HNSW efC=40 预期 ~20 min**。问题中实测的 100M HNSW 20 min 和 IVF-PQ 7+ h **完全对得上量级**。

### 2. PQ + IVF k-means 训练复杂度

[per concepts/product-quantization.md "三层结构 / 关键性质"; jegou-2011-pq §II.B + §IV]

PQ + IVFADC 训练三段：

- **Coarse quantizer 训练**：独立 k-means，k' ∈ [1k, 1M]，复杂度 ≈ samples · k' · D · iter
- **PQ codebook 训练**：m 个子量化器各跑 k-means，每子空间 256 centroids、D/m 维，复杂度 ≈ samples · 256 · (D/m) · iter · m = samples · 256 · D · iter
- **全量数据 assign + encode**：N · k' · D 距离计算（assign）+ N · m · (D/m) PQ encode（编码）

经典默认 `m=8, k*=256, k'=8192, w=8` [jegou-2011-pq §V.B]。

**关键观察**：k-means 训练时间 ∝ samples × k' × d × iter。如果训练 sample 数没截断（直接喂全量 N），训练时间 × N/sample_size 倍。

### 3. 为什么 IVF-PQ build 比 HNSW 慢这么多

HNSW build = N 个 vector 逐个 insert，每个 insert 做 efC 步的 graph descent + 加 M 条边；复杂度 ≈ N · M · efC · log(N) · d。CPU 上 HNSW 高度并行（per-vector insert 几乎独立），32 核近线性 scale。

IVF-PQ build 在两个方向比 HNSW 慢：
- **K-means 训练**是 iterative，必须 25 iter 串行（不能并行不同 iter），每 iter 内才并行 sample × centroid 距离
- **Coarse assign 阶段**（N · k' · D）= O(N · nlist) 是 brute-force scan all centroids per vector，没有"少做距离计算"的剪枝

→ 在 CPU 上 IVF-PQ 的 build cost 倍数对 HNSW 是 **5-15× 慢**（200M SIFT 实测 HNSW efC=40 42min vs IVFADC 11h = 15.7×）。

### 4. 为什么 wiki schema 允许 IVF-PQ 即使 build 慢

[per topics/gpu-vs-cpu-ann.md "反直觉之二"; malkov-2016-hnsw §5.4]

HNSW 200M SIFT 内存 64 GB，**Faiss PQ 23-30 GB**——PQ 内存压缩比 2-3×。在 1B+ 规模 HNSW 直接 OOM，PQ 路径不可绕开 [per benchmarks/hnsw-vs-faiss-200m-sift.md "核心 trade-off"]。所以 build time 慢 5-15× 是换 memory footprint 小 2-3× 的代价。

### 5. 修复方向（按 confidence 分）

**✅ Anchored 在 wiki 内：**

- 训练 sample 数应截断到 codebook 充分训练的子集（jegou-2011-pq 经典实验 `k*=256` 表示 256 centroids per subspace，不需要全量 N samples）
- HNSW 做 coarse quantizer 加速 assign 阶段 [per concepts/product-quantization.md "GPU 实现要点 / Open Questions": "IVFADC 的 coarse quantizer 用更优结构（如 IMI、HNSW-as-coarse-quantizer）"——已是工业界已知方向]
- GPU 加速 IVF-PQ build = Faiss-GPU SIFT1B vs CPU 8.5× faster [per benchmarks/faiss-gpu-sift1b-deep1b.md §6.4]（注意：是 search QPS 8.5×，不直接是 build time；论文 §6.5 给的 build 数字是 4×Titan X 跑 95M YFCC k-NN graph 35 min vs NN-Descent 128-CPU 集群 108.7 h）

**> [推测] 无 anchored 支持：**

- d=768 / nlist=10000 / ARM 鲲鹏 CPU 的具体 build time——wiki **无** 该配置实测
- "训练 sample = 1M 子集就够"的具体阈值——jegou-2011-pq 隐含但未做 sample size × recall ablation
- 训练样本截断的具体 wall-clock 收益——10× 还是 30× 减少 build time 取决于 nlist / d / 实现

### 6. 实测数据点（NEW，2026-05-20）

> [推测] 用户实测，机器 / d / 实现细节未完整记录到 wiki：
> - 100M vectors, HNSW (efC unknown, M unknown), CPU → **20 min**
> - 100M vectors, IVF-PQ (nlist=10000, m/nbits unknown, d unknown), CPU → **k-means 训练 7+ h, 整体未完成**

⚠️ 这两个数字**未充分参数化**，仅作 sanity check：与 malkov-2016 200M SIFT (128d) 数据点同量级。建议 follow-up 补全 d / 实现库 / 核数 / OPQ 是否启用，让该数据点可复现引用。

## Cited Pages

- [benchmarks/hnsw-vs-faiss-200m-sift.md](../benchmarks/hnsw-vs-faiss-200m-sift.md) — **核心 anchored baseline**: malkov-2016 Table 3, 200M SIFT CPU build time 全表
- [concepts/product-quantization.md](../concepts/product-quantization.md) — PQ + IVFADC 算法复杂度、典型参数
- [concepts/hnsw.md](../concepts/hnsw.md) — HNSW 构建复杂度
- [benchmarks/faiss-gpu-sift1b-deep1b.md](../benchmarks/faiss-gpu-sift1b-deep1b.md) — GPU vs CPU 加速倍数 anchored 数字
- [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md) — CPU 偏好 PQ 量化、GPU 偏好 brute-force 的总体框架
- [topics/index-selection.md](../topics/index-selection.md) — Faiss 索引选型决策树（N 规模 → 推荐索引）

## Follow-up Questions

- **高维（d ≥ 768）+ 大 nlist 的 IVF-PQ build time scaling**：wiki 内 zero coverage，主流 benchmark 都在 128-d SIFT。LLM embedding 时代 d=768/1024 普遍，但建索引 wall-clock 数据**没有**。建议下次 ingest Lance native PQ build benchmark / Faiss tutorials / 鲲鹏 ARM ANN 实测
- **K-means 训练 sample size × recall**：jegou-2011-pq 经典默认 `k*=256` 但未 sample size ablation；多大 sample 够、收益曲线长什么样——open
- **HNSW-as-coarse-quantizer for IVFADC 的 build/assign 加速实测**：concepts/product-quantization.md 标"部分工程化"（Faiss-GPU 已实现），但 CPU 上工业实证（节省多少 wall-clock）wiki 内无
- **ARM CPU vs x86 CPU 在 IVF-PQ build 上的差距**：NEON vs AVX-512 SIMD 差异对 k-means / PQ encode 的 wall-clock 影响——open

## Without-wiki baseline（作为 talk 演示对比）

未走 wiki 直接答时的回答片段：

> "100M IVF-PQ CPU build 应 30-60 分钟"（**错**——和 malkov-2016 anchored 量级差 10×）
> "8+ 小时不正常"（**错**——量级 normal，反向给出错误诊断）
> "GPU 快 10-15×"（**夸大**——anchored 是 8.5×, johnson-2017 §6.4）

→ **错误自信** 是 no-citation 模式的核心成本。这条 query 沉淀后下次同类问题 5 秒查到 anchored answer，避免重新现编。
