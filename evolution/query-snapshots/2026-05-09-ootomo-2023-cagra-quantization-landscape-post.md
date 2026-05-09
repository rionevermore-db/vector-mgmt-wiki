---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, concepts/cagra-graph.md, concepts/rabitq.md, concepts/product-quantization.md]
cited-count: 4
---

# Post-snapshot (ootomo-2023-cagra): quantization-landscape

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 不引入新 quantizer——但揭示 GPU memory bandwidth 对 quantization 的特殊偏好**。CAGRA 当前实现 **FP32 + FP16 dataset**，**未集成 PQ/RaBitQ**——论文 §V-E 列 quantization 为 future direction。**关键观察**：GPU memory bandwidth (1.5 TB/s) 让"无量化全精度"在 CAGRA 上仍可行（dataset 装得进 80 GB GPU memory）；CPU 或 disk 路径下"无量化"早已 OOM 不可行。**对 quantization 选择的影响**：GPU 路径下 quantization 优先目标从"减内存以装下"变为"提升 throughput"——FP16 (CAGRA default) 比 FP32 提速 30%; PQ/RaBitQ 集成后理论上让 CAGRA 上限从 ~100M 突破到 1B。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| Quantization use case | + streaming routing | **+ GPU bandwidth optimization (FP16)** |
| FP16 quantization | 完全空白 | **CAGRA default mode**（throughput +30%, recall 不退化） |
| GPU + PQ 集成 | Faiss-GPU IVFPQ 经典 | **CAGRA 未集成 PQ**（论文 §V-E future work） |
| GPU + RaBitQ 集成 | 完全空白 | **理论可行未实证**（CAGRA + RaBitQ 同 NVIDIA / NTU 不同生态） |

### CAGRA 在 quantization landscape 的位置（NEW）

[per systems/cagra.md "FP32 vs FP16" + ootomo-2023-cagra §V-E]

CAGRA 当前 quantization 选项：

| 选项 | Memory per vector | Throughput | Recall | CAGRA 实证 |
|---|---|---|---|---|
| FP32 | 4 byte/dim | reference | reference | ✓ |
| **FP16** | **2 byte/dim** | **+30%** | **不退化** | ✓ |
| PQ / OPQ | ~32 byte/vector | future | TBD | ✗ |
| RaBitQ | D bits/vector | future | TBD | ✗ |

**关键**：FP16 是 GPU 上 quantization 的"first-line"——而非 CPU 路径常见的 PQ。原因：GPU Tensor Core 原生 FP16 支持 + memory bandwidth bottleneck → FP16 "免费"提速 + 容量翻倍。

### CPU 与 GPU 的 quantization 偏好差异（NEW）

[per topics/gpu-vs-cpu-ann.md "关键 GPU 优化技术对照"]

| Quantization | CPU 路径 偏好 | **GPU 路径 偏好** |
|---|---|---|
| FP32 | OOM 必须压缩 | **可行 (in 80 GB HBM, dataset ≤25M for 768-d)** |
| FP16 | SIMD 部分支持 | **首选 (Tensor Core native, +30% throughput)** |
| PQ / OPQ | 必需 (内存敏感) | **可选** (future for >100M datasets) |
| RaBitQ | unbiased + sharp error bound 主导优势 | **未实证** |
| ScaNN anisotropic | MIPS 优化 | **未实证** |
| **GPU-specific 优化** | n/a | **warp splitting + shared memory hash table** |

→ CPU quantization 是"省内存"；**GPU quantization 是"增加 throughput / 突破 dataset size"**——目标不同。

### CAGRA + 量化的潜在收益（推断）

[per ootomo-2023-cagra §V-E + 推断]

理论上 CAGRA + quantizer 集成的影响：

| 集成方案 | 预期效果 |
|---|---|
| **CAGRA + FP16 (current)** | 容量 +100% (40M → 80M for 768-d), throughput +30% |
| **CAGRA + Faiss-GPU IVFPQ-style PQ** | 容量 +10×（80M → 800M），但 recall ceiling 受 PQ 失真 |
| **CAGRA + RaBitQ** | 容量 +2× (D bits = 一半 PQ default)，**recall 不退化** (sharp error bound)，**3× faster single distance** |
| CAGRA + DiskANN-style PQ + SSD rerank | GPU memory + SSD hybrid，1B+ 可行但 GPU SSD 延迟 trade-off 复杂 |

→ **CAGRA + RaBitQ** 是**理论上最优 GPU 量化**——但两个论文 (CAGRA 2024 + RaBitQ 2024) 同年同 NVIDIA / NTU 不同生态，相互不知；wiki 内 zero coverage frontier。

### Quantization landscape（updated with CAGRA）

| 方法 | 类型 | 主要 use case | 适合 hardware |
|---|---|---|---|
| Binary / SQ8 | scalar quantization | 内存极限 | CPU + GPU |
| FP16 | half precision | **GPU memory bandwidth + Tensor Core** | **GPU** |
| PQ / OPQ / LSQ | product quantization | 减内存 | CPU + Faiss-GPU IVFPQ |
| ScaNN anisotropic | score-aware | MIPS | CPU |
| VGPQ | PQ + Voronoi 几何 | OLAP | CPU (ADBV) |
| ACORN compression | graph edges | predicate-agnostic | CPU |
| RaBitQ | hypercube + random rotation | **unbiased + sharp bound** | CPU (untested on GPU) |
| **CAGRA-embedded FP16** | **half precision in graph traversal** | **GPU throughput** | **GPU only** |

### 已知盲区

- **CAGRA + RaBitQ 实证**：完全空白
- **CAGRA + PQ / OPQ 实证**：论文 §V-E 列 future
- **CAGRA + FP8 / INT8 / 更激进 quantization**：H100 / Blackwell Tensor Core 支持 FP8；未实证
- **PQ codebook 训练在 GPU 上的优化**：Faiss-GPU 早期工作；CAGRA 时代未更新
- **GPU + Disk hybrid quantization**：CAGRA + DiskANN-style PQ + SSD rerank 完全空白

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
