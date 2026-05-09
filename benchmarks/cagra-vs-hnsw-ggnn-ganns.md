---
title: CAGRA vs HNSW / GGNN / GANNS（GPU graph ANNS comprehensive benchmark）
type: benchmark
sources: [ootomo-2023-cagra]
related: [../systems/cagra.md, ../systems/faiss.md, ../systems/milvus.md, ../concepts/cagra-graph.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../topics/gpu-vs-cpu-ann.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md, ../benchmarks/hnsw-vs-faiss-200m-sift.md]
created: 2026-05-09
updated: 2026-05-09
---

# CAGRA vs HNSW / GGNN / GANNS

**TL;DR**: CAGRA ICDE 2024 论文 §V 在 7 个 dataset (SIFT-1M / GIST-1M / GloVe-200 / NYTimes / DEEP-1M/10M/100M) 上系统对比 CAGRA vs HNSW (CPU SOTA) + GGNN/GANNS (GPU SOTA) + NSSG (graph quality reference)，DGX A100 配置（A100 80GB GPU + EPYC 7742 64-core CPU）。**核心发现**：(a) **build time 比 HNSW 2.2-27× 快**，比 GPU baseline GGNN 1.1-31×、GANNS 1.0-6.1× 快；(b) **large-batch search (10K) 在 90-95% recall 比 HNSW 33-77× 快**，比 GPU baseline 3.8-8.8× 快；(c) **single-query 比 HNSW 3.4-53× 快**——这是 CAGRA 的关键独特优势（GPU graph ANN 多设计 for batch）；(d) **DEEP-100M scaling 仍 ~2× faster than HNSW**；(e) **rank-based reordering 比 distance-based 1.9× faster + DEEP-100M 唯一可行**（distance-based OOM）。

## 实验设置

[ootomo-2023-cagra §V-A]

### 硬件

| 组件 | 规格 |
|---|---|
| **GPU** | NVIDIA A100 80GB (full DGX A100) |
| **CPU** | AMD EPYC 7742 (64 cores, 128 threads) |
| **Memory layout** | dataset + graph 都在 GPU device memory |
| **构造阶段** | initial k-NN graph build on GPU；CAGRA optimization on CPU |

### 数据集

| Dataset | Dim (n) | Size (N) | Type | CAGRA degree d |
|---|---|---|---|---|
| SIFT-1M | 128 | 1M | float | 32 |
| GIST-1M | 960 | 1M | float | 48 |
| GloVe-200 | 200 | 1.18M | float | 80 |
| NYTimes | 256 | 290K | float | 64 |
| DEEP-1M | 96 | 1M | float | 32 |
| DEEP-10M | 96 | 10M | float | 32 |
| DEEP-100M | 96 | 100M | float | 32 |

→ degree d 各数据集不同——按 dataset 调（dim 高时 d 大）。

### Compared Methods

[ootomo-2023-cagra §V-A]

| Method | 实现 | 路径 |
|---|---|---|
| **CAGRA** | NVIDIA RAPIDS RAFT | **GPU graph (本论文)** |
| **GGNN** [Groh 2022] | original | GPU graph |
| **GANNS** [Yu 2022] | original | GPU graph (NSW + HNSW + k-NN) |
| **HNSW** [Malkov 2018] | libhnsw | **CPU graph SOTA** |
| **NSSG** [Fu 2022] | NSG variant | CPU graph quality reference |

### 关键参数

| Parameter | CAGRA value | 备注 |
|---|---|---|
| `d` (final out-degree) | 32-80 (per dataset) | uniform across nodes |
| `d_init` (initial k-NN graph degree) | 2d or 3d | NN-Descent input |
| `M` (internal top-M) | k 或更大 (调) | search 参数 |
| `p` (parents per iteration) | 1-4 (调) | larger p = more parallelism |
| Team size (warp split) | 4-32 (per dataset) | 96-d → 4-8; 960-d → 32 |
| Hash table size | 2^8 to 2^13 | shared memory budget |

## 主结果

### Result 1: Graph Construction Time（Fig 11）

[§V-A Q-C1]

| Method | SIFT-1M | GIST-1M | GloVe-200 | NYTimes |
|---|---|---|---|---|
| **CAGRA (A100)** | **14.5 s** | **28.9 s** | **25.1 s** | **7.6 s** |
| GGNN (A100) | 16.0 | 74.8 | 215.9 | 47.0 |
| GANNS (A100) | 14.6 | 229.7 | 151.9 | 35.8 |
| **HNSW (EPYC 7742 64-core)** | 32.3 | 901.1 | 172.2 | 226.3 |
| NSSG (EPYC 7742) | 120.5 | 691.6 | 981.7 | 311.7 |

**Speedup vs HNSW**: 2.2× SIFT, **31× GIST**, 6.9× GloVe, **30× NYTimes**——平均 ~10×。

**Speedup vs GPU baseline**: 1.0-31× (GGNN/GANNS 都比 CAGRA 慢)。

### Result 2: Construction time breakdown（Fig 11 + Fig 15）

[§V-A]

CAGRA 三阶段时间：
1. **Initial k-NN build (NN-Descent on GPU)**: ~50-70% of total
2. **Graph optimization (rank-based reordering + reverse edge)**: ~20-30%
3. **Indexing**: ~5-10%

DEEP scaling [Fig 15]:
- DEEP-1M: 14.6 s (CAGRA) vs 27.3 s (HNSW)
- DEEP-10M: 130.4 s (CAGRA) vs 236.5 s (HNSW)
- **DEEP-100M**: **1305.6 s (CAGRA)** vs **2623.3 s (HNSW)** = **2× faster**

→ 大数据集时 CAGRA 优势收窄但仍 2×。

### Result 3: Rank-based vs Distance-based reordering（Fig 4）

[§III-C Q-A2]

| Dataset | Rank-based time | Distance-based time |
|---|---|---|
| SIFT-1M | reference | 1.9× |
| GIST-1M | reference | 1.5× |
| GloVe-200 | reference | 1.6× |
| NYTimes | reference | 1.7× |
| DEEP-10M | reference | 2.1× |
| **DEEP-100M** | **可行** | **OOM** |

→ Rank-based **1.5-2.1× faster**；DEEP-100M 唯一可行（distance-based 需 N × d_init distance table 装不下）。

### Result 4: Recall comparison rank vs distance（Fig 5）

[§III-C Q-A3]

QPS-recall 曲线 SIFT/GIST/GloVe/NYTimes：rank-based 与 distance-based 几乎重合——**recall 完全 comparable**。

→ Rank-based 是 **strict win**（更快 + 同 recall）。

### Result 5: Search Performance Large-Batch（Fig 13）

[§V-C Q-C3, batch size = 10K]

90-95% recall@10 范围（all 4 datasets SIFT/GIST/GloVe/NYTimes）：

| Method | QPS at 95% recall (SIFT-1M) |
|---|---|
| **CAGRA (FP16, A100)** | **highest** |
| CAGRA (FP32, A100) | second |
| GGNN (A100) | mid (3.8-8.8× slower than CAGRA) |
| GANNS (A100) | similar to GGNN |
| **HNSW (CPU 64-thread)** | **33-77× slower than CAGRA** |
| NSSG (CPU) | similar to HNSW |

**FP16 mode**: CAGRA FP16 比 FP32 快 ~30%（memory bandwidth bound），recall 不退化。

### Result 6: Search Performance Single-Query（Fig 14）

[§V-D Q-C4]

95% recall@10 single-query latency (single CTA / multi-CTA dual mode dispatched):

| Method | Single-query throughput |
|---|---|
| **CAGRA** | reference baseline |
| HNSW (CPU) | **3.4-53× slower** at 95% recall |
| GGNN | poor (designed for batch) |
| GANNS | poor (designed for batch) |

→ CAGRA 是**唯一支持 small-batch / single-query 的 GPU graph ANN**——这是它与 GGNN/GANNS 的关键差异。multi-CTA mode 让单 query 可以利用整 GPU 多 SM。

### Result 7: Graph Quality Comparison（Fig 12）

[§V-B Q-C2]

CAGRA graph 装入 NSSG search 实现（替换 NSSG 自家 graph）vs NSSG graph 装入 NSSG search：

| Dataset | CAGRA graph + NSSG search | NSSG graph + NSSG search |
|---|---|---|
| SIFT/GIST/GloVe/NYTimes | **comparable** | reference |

→ CAGRA graph **quality 与 NSSG comparable**（NSSG 是 ANNS 最高 quality CPU graph 之一）+ **build 5-30× faster**。

### Result 8: Team Size Sensitivity（Fig 8）

[§IV-D Q-B1]

| Dataset | Optimal team size | 备注 |
|---|---|---|
| **DEEP-1M (96-d)** | **4 or 8** | 低维 → split for parallelism |
| GIST-1M (960-d) | **32 (no split)** | 高维 → single team uses all warp |
| GloVe-200 | 8-16 | mid |
| NYTimes (256-d) | 16 | mid |

→ **Team size 自适应 dimensionality**——logic: load instruction (128-bit) × team size = vector dim × 4 byte。

### Result 9: Forgettable Hash Table（Fig 9）

[§IV-D Q-B2]

DEEP-1M / GloVe-200: forgettable hash (shared mem) vs standard hash (device mem):

| Recall | Forgettable QPS | Standard QPS |
|---|---|---|
| 0.85 | high | high |
| 0.95 | **higher** | high |
| 0.99 | **higher** | mid |

→ Forgettable hash **comparable or higher**——shared memory access 比 device memory 显著更快，trade-off acceptable。

### Result 10: Single-CTA vs Multi-CTA（Fig 10）

[§IV-D Q-B3]

DEEP-1M / GloVe-200, single-query vs large-batch (10K):

| Mode | Single-query | Large-batch |
|---|---|---|
| Single-CTA | low (CTA underutilized) | **highest** |
| Multi-CTA | **highest** (multiple CTAs share work) | mid |

→ CAGRA framework **dispatch automatically** based on batch size + recall target.

## 可信度评估

### 实验设计

- ✓ 7 datasets covering low-dim (SIFT 128) / high-dim (GIST 960) / multiple data types
- ✓ 4 GPU baselines + 1 CPU SOTA + 1 graph quality reference
- ✓ Both build time + search throughput evaluated
- ✓ Single-query + large-batch both测
- ✓ Open source NVIDIA RAFT
- ✓ Hardware spec 公开（DGX A100 + EPYC 7742）
- ✗ **NVIDIA team 自评** — baseline 选择 favorable 是潜在 bias
- ✗ **未对比 Faiss-GPU IVFPQ** — 不同 algorithm 路径但工业 GPU baseline；遗漏值得对比

### 复现难度

- NVIDIA RAPIDS RAFT 开源（Apache-2.0）
- DGX A100 是 enterprise 级 hardware，但 A100 单卡（在 cloud 可访问）足以复现
- HNSW (libhnsw) 开源
- GGNN / GANNS / NSSG 也开源（论文 references）
- 7 datasets 全公开 (texmex.irisa.fr)

### 偏向

- **NVIDIA-favoring**：作者全 NVIDIA；GPU vs CPU baseline 对比时 GPU hardware 是 SOTA (A100 80GB) but CPU 是中端 (EPYC 7742 64-core)——不是 absolute 公平比较，但**反映典型 production 配置**
- **算法选择 bias**：作者选 NSSG 作 graph quality reference 而非 HNSW——NSSG 比 HNSW 在 ANNS quality 略好，是合理选择；但 HNSW 是工业 default
- **GPU memory budget bias**：100M-1B 才是真实工业问题；论文上限是 100M——超出 single A100 80GB 时 CAGRA 仍可行性未实证
- **Multi-GPU 与 streaming 缺**——CAGRA 的两个最大开放问题

### 数据集偏向

- 6/7 dataset 在 1M 或更小（DEEP-100M 是最大）
- SIFT/GIST/GloVe/DEEP 都是 image / text vector benchmark——经典 ANN community datasets
- 缺真实 production workload（recommendation / RAG embeddings）

## Open Questions

- **CAGRA vs Faiss-GPU IVFPQ head-to-head**：CAGRA 论文未对比；wiki 内推论 CAGRA 高 recall + Faiss-GPU 内存效率
- **Multi-GPU sharding 详细协议**：论文 §IV-C 提及但 1 段；详细未实证
- **CAGRA + PQ / RaBitQ memory compression**：减少 GPU memory footprint 让超 GPU 显存的 dataset 可行；论文 §V-E 列 future
- **CAGRA + streaming insert/delete**：fixed-degree graph 的 streaming 比 variable-degree HNSW/NSG 难——完全 open
- **CAGRA + filter / multi-vector / range / Join**：完全 open
- **DEEP-1B / SIFT-1B 实测**：缺 multi-GPU 路径
- **CAGRA + DiskANN-style hybrid**：GPU memory 不够时 fallback 到 SSD
- **embedding model 升级**：所有 wiki source 一致——zero coverage
