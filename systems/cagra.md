---
title: CAGRA / NVIDIA RAPIDS RAFT（GPU-native graph ANN library）
type: system
sources: [ootomo-2023-cagra]
related: [faiss.md, milvus.md, diskann.md, freshdiskann.md, ../concepts/cagra-graph.md, ../concepts/warpselect.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/vamana.md, ../concepts/product-quantization.md, ../topics/gpu-vs-cpu-ann.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/cagra-vs-hnsw-ggnn-ganns.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md]
created: 2026-05-09
updated: 2026-05-09
---

# CAGRA / NVIDIA RAPIDS RAFT

**TL;DR**: NVIDIA 在 ICDE 2024 发布的**首个 GPU-native graph-based ANN system**，集成于 [NVIDIA RAPIDS RAFT library](https://github.com/rapidsai/raft)。CAGRA = **C**uda **A**nns **GR**A**ph**——基于 [CAGRA graph 算法](../concepts/cagra-graph.md)（fixed out-degree + non-hierarchical + directional + rank-based reordering）+ 4 个 GPU search 优化（warp splitting / forgettable hash table / 1-bit parented node / single-CTA + multi-CTA dual mode）。**实测 large-batch search 比 HNSW (CPU 64-core) 33-77× 快**；**graph build 比 HNSW 2.2-27× 快**；**single-query 3.4-53× 快于 HNSW**。**Production 集成**：[Milvus](./milvus.md) v2.6.x **GPU_CAGRA** 索引直接基于 RAFT；[Faiss](./faiss.md) 当前未集成（论文 §A.3 emerging direction，**ingest 后关闭此 frontier**）。NVIDIA 认为 CAGRA 是 RAPIDS 生态在 vector search 的旗舰产品——open source Apache-2.0。

## 与 wiki 现有 GPU ANN 系统的定位差异

[per topics/gpu-vs-cpu-ann.md, ootomo-2023-cagra §II-C]

之前 wiki 内 GPU ANN 路径：

| | [Faiss-GPU IVFADC](./faiss.md) | **CAGRA** |
|---|---|---|
| Algorithm 路径 | IVF + PQ | **graph-based**（NSW/NSG-style）|
| 论文年份 | 2017 NeurIPS | **2024 ICDE** |
| 团队 | Meta Facebook | **NVIDIA** |
| GPU 优化粒度 | warp + shared memory PQ LUT | warp + shared memory hash table + warp-splitting + bitonic sort |
| 适合 batch | large-batch (≥1000) | **single-query + large-batch (1-10K)** |
| 适合 dataset size | up to GPU memory | up to GPU memory |
| Recall ceiling | ~70% (受 IVF + PQ 约束) | **~99%+ (graph search high recall)** |
| Production 集成 | Faiss release | **NVIDIA RAPIDS RAFT** |
| Vector DBMS 集成 | Milvus IVF*GPU* | **Milvus GPU_CAGRA** |

**核心论点**：之前 GPU ANN 限于 IVF + PQ 路径；CAGRA 把 graph-based ANN 完整带到 GPU——**recall ceiling 显著提升 + single-query 成为可能**。

## 架构图

[ootomo-2023-cagra §IV + Fig 7]

```
┌────────────────────────────────────────────────┐
│  Host (CPU)                                    │
│  ├─ Initial k-NN graph build orchestration    │
│  ├─ CAGRA graph optimization (rank-based)     │
│  └─ Query batch dispatch                       │
├────────────────────────────────────────────────┤
│  GPU Device Memory                             │
│  ├─ Dataset vectors (FP32 / FP16)             │
│  ├─ CAGRA graph (fixed degree d adjacency)    │
│  └─ Hash table (multi-CTA mode)               │
├────────────────────────────────────────────────┤
│  GPU Shared Memory (per CTA)                   │
│  ├─ Internal top-M list                       │
│  ├─ Candidate list (length p × d)             │
│  ├─ Hash table (single-CTA mode, ≤4KB)        │
│  └─ Bitonic sort buffers                      │
└────────────────────────────────────────────────┘

Search Dispatch:
  if batch_size > #SMs OR M > 512: single-CTA  (large batch)
  else: multi-CTA  (small batch / high recall)
```

## 数据流 / 控制流

### Index Build

[ootomo-2023-cagra §III-B]

```
Input: dataset D in GPU memory

1. Build initial k-NN graph on GPU
   - Use NN-Descent algorithm
   - k = d_init = 2d or 3d (where d = final degree)
   - Sort each node's neighbor list by distance

2. CAGRA graph optimization (sequential or parallel-friendly):
   a) Rank-based reordering + pruning
      - Count detourable routes per edge using rank (not distance)
      - Sort edges by detourable count, keep top d
   b) Reverse edge addition
      - Add reverse edges, prune by RNG rule
      - Merge: d/2 children + d/2 reversed
   
3. Output: CAGRA graph in GPU device memory
```

实测 [Fig 11]: SIFT-1M 14.5 s; GIST-1M 28.9 s; DEEP-100M **1305.6 s**（A100）。

### Search (Multi-CTA mode for small batch)

```
For each query q:
  Launch multiple CTAs sharing one query's work:
    Each CTA:
      ⓪ Random sample p × d candidate nodes
      Loop:
        ① Bitonic sort top-M from buffer
        ② Graph traversal: pick top-p parents → fetch d neighbors
        ③ Distance compute for first-time candidates
      Aggregate top-K from all CTAs
```

### Search (Single-CTA mode for large batch)

```
For batch of queries (size ≥ 100):
  Each query → one CTA
    Same algorithm but:
    - Hash table in shared memory (forgettable)
    - team-size sized warp splitting for distance compute
```

## 关键设计决策

### 1. NVIDIA RAPIDS RAFT integration（§I + §VI）

[ootomo-2023-cagra §I 末尾, §VI]

CAGRA = **Cuda Anns GRAph based**——专为 NVIDIA GPU SIMT 模型 + CUDA programming model 设计。集成于：

| RAPIDS 组件 | 关系 |
|---|---|
| **RAFT (RAPIDS AI Framework Toolkit)** | CAGRA 的 reference implementation library |
| cuML | RAFT 的 ML application layer |
| cuVS | Vector search-specific NVIDIA library（some functionality refactored from RAFT） |

→ **CAGRA 不是 isolated paper**——是 NVIDIA 全家桶（RAPIDS）的 vector search piece。

### 2. Production 集成路径

[per systems/milvus.md + 推断]

**已集成**：
- **[Milvus](./milvus.md) v2.6.x GPU_CAGRA 索引** — 直接基于 RAFT
- NVIDIA cuVS / RAPIDS 用户 (RAPIDS-based ML pipeline)

**未集成（frontier）**：
- **[Faiss](./faiss.md)** — 论文 §A.3 提及；当前 release 不集成（**ingest 后关闭 Faiss frontier flag**）
- DiskANN-style on-GPU — 不直接（DiskANN 是 SSD-resident）
- VBASE / PASE — query engine layer，与 CAGRA 正交但未实证联合

### 3. FP32 vs FP16 dataset support

[ootomo-2023-cagra §V-C]

CAGRA 支持 FP32 dataset 与 **FP16-converted dataset** 两种模式：
- FP32: full precision, default
- FP16: half precision, **更快 large-batch throughput**（memory bandwidth bound）, recall 不退化

实测 [Fig 13]: FP16 模式比 FP32 模式快 ~30%（SIFT/GloVe/NYTimes large-batch），recall 几乎相同。

### 4. Single-CTA vs Multi-CTA dual implementation

[ootomo-2023-cagra §IV-C + Fig 7]

CAGRA framework **runtime dispatch** between two implementations:

| | Single-CTA | Multi-CTA |
|---|---|---|
| 适合 | large batch (≥100) | small batch / high recall |
| Hash table | shared memory + forgettable | device memory + standard |
| Per query | 1 CTA | multiple CTAs (typically ~#SMs) |

→ **CAGRA 自动 dispatch**——同一 framework handle 大小 batch 两种 workload。这是与 GGNN（仅 large-batch friendly） / GANNS（同样 batch-only）的关键差异。

## 实测亮点

[详见 benchmarks/cagra-vs-hnsw-ggnn-ganns.md](../benchmarks/cagra-vs-hnsw-ggnn-ganns.md)

- **Build time vs HNSW (CPU 64-core)**: **2.2-27× faster** on A100 GPU
- **Build time vs GGNN (GPU)**: 1.1-31× faster
- **Build time vs GANNS (GPU)**: 1.0-6.1× faster
- **Large-batch search (10K) at recall 90-95%**: **33-77× faster than HNSW**, **3.8-8.8× faster than GGNN/GANNS**
- **Single-query at recall 95%**: **3.4-53× faster than HNSW**
- **DEEP-100M build**: 1305.6 s (CAGRA) vs 2623.3 s (HNSW)
- **DEEP-100M search**: similar trends (~2× CAGRA over HNSW)
- **FP16 mode**: ~30% additional throughput, no recall loss
- **Graph quality**: comparable to NSSG (highest CPU graph quality for ANNS)

## Scale 边界

[ootomo-2023-cagra §V-E + recommended NVIDIA hardware]

| 配置 | 数据 | 实测 / 推断 |
|---|---|---|
| **NVIDIA A100 80GB** | DEEP-100M (96-d float32) | **build 1305 s + search OK** |
| A100 80GB | 100M × 768-d float32 | ~300 GB → **OOM**（需 multi-GPU 或 FP16/PQ） |
| Multi-GPU sharding | 1B+ | 论文 §IV-C 提及但未深入实证 |
| **GPU memory + PQ compression** | future work | 论文 §V-E 列为 future |

> **wiki 解读**：CAGRA 受 GPU memory 限制——**单 A100 ~100M float32 OK**；更大需 multi-GPU 或 PQ 压缩或 FP16。这与 [DiskANN single 1B SIFT 64 GB CPU](./diskann.md) / [SPANN Bing 几千亿](./spann.md) 是不同 budget——GPU 路径更快但内存更小。

## 与 wiki 已有系统的对比

### 与 [Faiss-GPU IVFADC](./faiss.md)（同年代不同算法路径）

[详见 benchmarks/faiss-gpu-sift1b-deep1b.md vs benchmarks/cagra-vs-hnsw-ggnn-ganns.md]

| | Faiss-GPU IVFADC | **CAGRA** |
|---|---|---|
| 算法 | IVF + PQ | **graph-based (NSW/NSG-style)** |
| Recall ceiling (no rerank) | ~70% (PQ 失真) | **~99%+ (graph high recall)** |
| Single-query | 不友好（设计 for batch） | **友好（multi-CTA mode）** |
| Large-batch SIFT1B QPS | 高（IVFPQ + WarpSelect 优化） | 高 (1B 需 multi-GPU) |
| Build time SIFT1M | seconds (IVF train fast) | **14.5 s** (NN-Descent + optimization) |
| Memory footprint | small (PQ codes ~32 byte) | larger (full graph + adjacency) |
| Production | Faiss release | **RAPIDS RAFT** |

→ Faiss-GPU 偏 PQ + 内存效率；CAGRA 偏 graph + recall ceiling。两者 **complementary**——相同 GPU 上不同 ANN 路径。

### 与 [DiskANN](./diskann.md) / [FreshDiskANN](./freshdiskann.md)（CPU 路径）

CAGRA 与 DiskANN/FreshDiskANN 走**完全不同 hardware path**：
- DiskANN: CPU + SSD（disk-resident，1B SIFT 64GB RAM）
- FreshDiskANN: CPU + SSD + DRAM（streaming-ready）
- CAGRA: GPU device memory（限于 ~100M-1B per GPU）

**目标 workload 不同**——DiskANN 是 cost-effective billion-scale，CAGRA 是 highest-throughput within GPU memory。

理论上 **CAGRA + Vamana on disk hybrid** logical：CAGRA in-memory graph + DiskANN-style PQ + SSD re-rank。但论文未涉及；CAGRA 仅 in-memory。

### 与 [Milvus](./milvus.md) GPU_CAGRA 索引

[per systems/milvus.md v2.6.x indices]

Milvus v2.6.x docs 列 GPU_CAGRA 索引——**实质上是 NVIDIA RAFT integration**（Zilliz 与 NVIDIA 合作）。

| | Milvus GPU_CAGRA | RAFT CAGRA standalone |
|---|---|---|
| Algorithm | 同（来自 RAFT） | 同 |
| 数据流 | Milvus segment 模型 | RAFT API 直接 |
| Production-readiness | Milvus tested | RAPIDS tested |
| Update model | per-segment static (Milvus default) | static |

→ Milvus 是 CAGRA 主要 production deployment 路径——Zilliz 团队（[Manu/Milvus](./milvus.md) / [Starling](./starling.md) 作者）将 CAGRA 列为 GPU 主索引。

### 与 [GGNN / GANNS / SONG](../concepts/cagra-graph.md "前作 GPU graph ANNS")

CAGRA 论文 §II-C 比较前作：
- **SONG** [Zhao 2021]: 首个 GPU graph ANNS，适配 NSW；10-180× over single-thread CPU
- **GGNN** [Groh 2022]: GPU-friendly k-NN graph + fast construction; outperforms SONG large-batch
- **GANNS** [Yu 2022]: NSW + HNSW + k-NN GPU；比 SONG 快但比 CAGRA 慢

CAGRA 在 build time + search throughput **systematically dominate** all 3 GPU baselines on 6 datasets。

## 生产案例

- **NVIDIA RAPIDS RAFT**: [github.com/rapidsai/raft](https://github.com/rapidsai/raft)，Apache-2.0
- **NVIDIA cuVS**: vector search refactor of RAFT，独立 library
- **Milvus v2.6.x GPU_CAGRA index**: production-ready
- **BIGANN'21 NeurIPS challenge**: NVIDIA 团队（同作者）参赛使用 CAGRA 思路

> **wiki 解读**：CAGRA 是**NVIDIA 在 vector search 领域的旗舰产品**——RAPIDS / cuVS / Milvus 集成路径明确。这与之前 wiki 内 [Faiss-GPU Johnson 2017] 的 Meta Facebook 角色 parallel——两个 GPU 厂商生态各自 ANN 实现。

## Open Questions

- **Faiss + CAGRA 集成**: Faiss 论文 §A.3 提为 emerging direction；当前 Faiss release 不集成 RAFT。集成是 logical work，但牵涉 Meta vs NVIDIA 生态边界
- **Multi-GPU sharding 详细协议**: 论文 §IV-C 提及但仅 1 段；具体 graph 切分 + cross-GPU search aggregation + load balance 未深入
- **CAGRA + PQ / FP16 / [RaBitQ](../concepts/rabitq.md)**: §V-E 列 quantization 为 future direction；当前只有 FP32/FP16
- **CAGRA + streaming insert/delete (FreshCAGRA)**: 论文不涉及；fixed-degree graph 的 streaming 比 variable-degree HNSW/NSG 难（每 insert 必须 prune 不能 grow degree）
- **CAGRA + filter / multi-vector / range / Join**: 论文仅 pure ANNS；与 wiki 内 [filter-aware](../concepts/filtered-vamana.md) / [iterator + RM](../concepts/relaxed-monotonicity.md) 联合 logical
- **CAGRA + iterator + VBASE**: 理论上 CAGRA 满足 RM；但 GPU batch + CPU iterator 接口 fit 是开放
- **CAGRA + DiskANN-style SSD hybrid**: 大 dataset (1B+) 必须 PQ + SSD 或 multi-GPU；论文未实证
- **NVIDIA RAFT 与 cuVS 关系**: 两个 NVIDIA library 都有 vector search；明确分工 wiki 未深入
- **CAGRA 与 Faiss-GPU IVFPQ 同 hardware 直接对比**: 论文未对比（不同 algorithm 路径）；wiki 内推论 CAGRA 高 recall + Faiss-GPU 大数据
- **embedding model 升级**: 与 wiki 全 frontier 一致——zero coverage

Cited by: [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md)
