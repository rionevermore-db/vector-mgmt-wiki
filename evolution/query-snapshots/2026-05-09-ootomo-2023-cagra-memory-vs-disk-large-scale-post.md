---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, systems/diskann.md, systems/spann.md, concepts/cagra-graph.md, topics/gpu-vs-cpu-ann.md]
cited-count: 5
---

# Post-snapshot (ootomo-2023-cagra): memory-vs-disk-large-scale

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 给 memory-vs-disk landscape 添加 GPU memory 维度**——之前 wiki 内存-disk 分类 (DRAM / SSD / GPU HBM) 中 GPU 路径仅 Faiss-GPU IVFPQ；CAGRA 是首个 GPU-native graph ANN，让 GPU memory 的 high recall search 成为可能。**核心 trade-off**：GPU memory ~80 GB (A100) 是关键 budget——CAGRA single-A100 大约支持 **100M 向量 (96-d float32)**；超过需 multi-GPU sharding 或 PQ/FP16 压缩。**对 1B+ workload 的影响**：CAGRA 不直接挑战 [DiskANN single 1B SIFT 64GB CPU](../../systems/diskann.md) / [SPANN Bing 几千亿](../../systems/spann.md) / [SPFresh 1B 4GB](../../systems/spfresh.md)——这些 disk path 经济性更高；CAGRA 适合 **<100M dataset 但需要极致 throughput / single-query latency** 的场景。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| 内存层级覆盖 | DRAM + SSD (cluster + graph paths) | **+ GPU HBM (CAGRA)** |
| Single-machine 1B 实证 | DiskANN / SPANN / SPFresh / FreshDiskANN | **+ CAGRA single A100 ~100M (DEEP-100M)** |
| GPU graph 路径 | 完全空白 | **首次明确（CAGRA）** |
| Recall ceiling 维度 | DiskANN / SPANN ~95-99% | **+ CAGRA 99%+ via graph high recall** |

### 路线对比表（updated with CAGRA）

| 路线 | 数据驻留 | static / streaming | quantization | 1B memory budget | scale 实证 |
|---|---|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | static | ✗ | OOM | 1B OOM |
| DiskANN | DRAM (PQ) + SSD | static | PQ | ~64 GB | 1B SIFT |
| SPANN | DRAM (centroids) + SSD | static | ✗ | ~32 GB | 1B+ Bing |
| SPFresh + LIRE | DRAM + SSD | streaming (cluster) | ✗ | ~4 GB | 1B SIFT 100 days |
| FreshDiskANN + FreshVamana | DRAM + SSD | streaming (graph) | PQ | ~128 GB | 800M SIFT week-long |
| RaBitQ + IVF (RAM only) | DRAM | static | RaBitQ | ~128 GB | 6 dataset 2.34M |
| Starling (segment-level disk) | DRAM + SSD | static (Milvus segment) | PQ | per-segment 2GB+10GB | BIGANN 1B 31 segs |
| Faiss-GPU IVFPQ | GPU HBM | static | PQ | per-GPU 16-80 GB | 1B SIFT 1 GPU |
| **CAGRA (NEW)** | **GPU HBM** | **static** | **none (FP32) / FP16 / future PQ** | **per-GPU 16-80 GB** | **DEEP-100M single A100** |

### CAGRA 在 memory hierarchy 的位置（NEW）

[per concepts/cagra-graph.md + ootomo-2023-cagra §V-E]

| Memory tier | Capacity | Bandwidth | CAGRA 角色 |
|---|---|---|---|
| GPU registers | ~256 KB/CTA | TB/s | bitonic sort buffer |
| GPU shared memory | 48 KB/CTA (A100) | TB/s | **forgettable hash table + internal top-M list** |
| GPU device memory (HBM) | 80 GB (A100) | 1.5 TB/s | **dataset + CAGRA graph** |
| CPU DRAM | 100s GB | 50 GB/s | host pre-processing |
| SSD | TBs | 5 GB/s | **CAGRA 不用** (in-memory only) |

→ CAGRA **完全 GPU memory-resident**——这是与 DiskANN / SPANN / FreshDiskANN 的根本架构差异。

### CAGRA 的 GPU memory 上限（NEW）

[per ootomo-2023-cagra §V-E + 推断]

A100 80GB GPU memory 容量估算：

| Vector type | Per-vector size | A100 单 GPU 容量 |
|---|---|---|
| 96-d float32 | 384 byte (raw) + ~256 byte (graph adjacency) = ~640 byte | **~125M vectors** |
| 128-d float32 (SIFT) | 512 + 256 = 768 byte | **~100M vectors** |
| 768-d float32 (LLM embedding) | 3072 + 256 = 3328 byte | **~24M vectors** |
| 1024-d float32 | 4096 + 256 = 4352 byte | **~18M vectors** |

→ **CAGRA single-A100 大约 100M vectors (96-d)** 是论文 DEEP-100M 的实证；高维数据更受限。

**FP16 模式**: 容量翻倍 (~200M for 96-d)，throughput +30%，recall 不退化——是常见 production 配置。

**Multi-GPU sharding**: 论文 §IV-C 提及但仅 1 段；详细未实证。理论上 8 × A100 → 1B 可行，但 cross-GPU search aggregation 协议是 open work。

### CAGRA 不竞争的 workload（NEW）

| Workload | 推荐路径 | CAGRA 适合性 |
|---|---|---|
| **1B+ + 内存预算严 + cost-sensitive** | SPFresh / SPANN / DiskANN | ✗（GPU memory 不够） |
| **1B+ + 多 segment per machine** | Milvus + Starling per segment | ✗ |
| **Streaming 1B+** | FreshDiskANN / SPFresh | ✗ (CAGRA static) |
| **Filter-heavy** | FilteredVamana / ACORN | ✗ (CAGRA pure ANNS) |
| **Multi-column / Range / Join** | VBASE | ✗ (CAGRA TopK only) |
| **Dataset ≤100M + 高 recall + GPU 可用** | **CAGRA** | **✓ ✓ ✓** |
| **Dataset ≤100M + low single-query latency** | **CAGRA multi-CTA** | **✓ ✓ ✓** |
| **Dataset ≤100M + 极致 large-batch throughput** | **CAGRA single-CTA + FP16** | **✓ ✓ ✓** |

→ CAGRA 是**特定 niche 的 dominator**，不是通用 vec DB 解。

### 与 [Faiss-GPU IVFPQ](../../systems/faiss.md) 的对比（NEW）

[per benchmarks/faiss-gpu-sift1b-deep1b.md vs benchmarks/cagra-vs-hnsw-ggnn-ganns.md]

| | Faiss-GPU IVFPQ (2017) | **CAGRA (2024)** |
|---|---|---|
| 算法 | IVF + PQ | graph-based |
| Recall ceiling | ~70% (PQ 失真) | **~99%+** |
| Memory footprint | small (PQ ~32 byte) | larger (graph + full vector) |
| Single-query | 不友好 | **友好** (multi-CTA mode) |
| Build time SIFT1M | seconds | **14.5 s** |
| 1B SIFT (1 GPU) | ✓ via PQ | ✗ (memory budget 不够) |

→ **Faiss-GPU 偏内存效率 + 大数据**；**CAGRA 偏 recall ceiling + small dataset**。两者**complementary**，不是替代关系。

### 已知盲区（仍未覆盖）

- **CAGRA + multi-GPU 1B+ sharding 实证**：论文未深入
- **CAGRA + PQ / RaBitQ memory compression**：论文 §V-E 列 future
- **CAGRA + DiskANN-style hybrid (GPU + SSD)**：完全空白
- **GPU streaming graph (FreshCAGRA)**：完全空白
- **CAGRA + filter / multi-vector**：完全空白
- **NVIDIA H100 / B100 上 CAGRA 性能**：H100 HBM3 bandwidth 比 A100 高 2-3×；论文止于 A100
- **CAGRA vs Faiss-GPU IVFPQ head-to-head**：完全空白（两个论文同 NVIDIA 但不同团队 + 不同算法路径）

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
