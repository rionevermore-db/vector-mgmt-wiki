---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/cagra-graph.md, systems/cagra.md, topics/gpu-vs-cpu-ann.md]
cited-count: 5
---

# Post-snapshot (ootomo-2023-cagra): hnsw-vs-nsg-selection

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA ingest 把 HNSW vs NSG 决策维度从 CPU-only 扩展到 CPU+GPU 双 hardware**——之前 wiki 把 HNSW/NSG 视作 CPU 算法 + 在 disk-resident segment 场景 (Starling) 评估。CAGRA 揭示**HNSW 多层结构 + NSG 单层 + Vamana α-RNG 都是 CPU-friendly first principles 的产物**——GPU 重新设计后 graph 形态完全不同（fixed degree + non-hierarchical + rank-based reordering）。**对决策的影响**：HNSW vs NSG 选择**仅适用 CPU**；**GPU 路径直接选 CAGRA**——HNSW/NSG 在 GPU 上是 GGNN/GANNS-style 移植，被 CAGRA 系统击败。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| HNSW α=1 streaming 失败 | 首次明确 | 不变 |
| NSG α=1 streaming 失败 | 同上 | 不变 |
| HNSW vs NSG 决策维度 | CPU + disk-resident segment + streaming | **+ CPU vs GPU hardware path** |
| GPU graph 算法选择 | 完全空白 | **CAGRA: fixed degree + non-hierarchical + rank-based** |
| HNSW/NSG 在 GPU 的状态 | 完全空白 | **CAGRA paper 揭示 GGNN/GANNS-style 移植被 CAGRA 系统击败** |

### CPU vs GPU graph 算法的"双路径"（NEW）

[per topics/gpu-vs-cpu-ann.md "CPU graph + GPU graph 的关系" + concepts/cagra-graph.md "三大设计特性"]

**HNSW 多层 + NSG 单层 + Vamana α-RNG 都是 CPU-friendly first principles**：
- Variable degree → 减少不必要 distance computation（CPU bottleneck）
- Hierarchy → 加速 greedy descent（CPU sequential）
- α-RNG → preserve navigability（CPU 需要不要太稀疏）

**GPU 上的 first principles 完全不同**（CAGRA design）：
- Fixed degree → uniform parallelism（避免 warp divergence）
- Non-hierarchical → random sampling（GPU 高并行替代 hierarchical descent）
- Rank-based reordering → 不需 distance 重算（GPU memory bandwidth）

→ **HNSW vs NSG vs Vamana 是 CPU 内部选择**；**CPU vs GPU graph (HNSW vs CAGRA) 是 hardware 路径选择**。

### HNSW vs NSG 完整决策表（updated）

| 工程考量 | HNSW (CPU) | NSG (CPU) | **CAGRA (GPU)** |
|---|---|---|---|
| 构建成本 | 高 | 更低 | **极低 GPU**（A100 比 64-core 2.2-27× 快） |
| 单点查询性能 | 中 | 更高 | **极快**（multi-CTA mode） |
| Million-scale dataset 击败 | NSG 击败 | NSG 胜 | **CAGRA 同 NSG quality + 远更快** |
| 增量 add-only | ✓ | ✗ | ✗ (static graph) |
| Streaming insert+delete | ✗ (α=1 fail) | ✗ (α=1 fail) | ✗ (open work) |
| Production 实证（in-memory） | hnswlib / Faiss / Milvus / Pinecone | Taobao 2B (static) | **NVIDIA RAPIDS RAFT / Milvus GPU_CAGRA** |
| Predicate-agnostic filter (HCPS) | ✓ via ACORN | ✗ | ✗ (open) |
| Iterator + RM 集成 | ✓ via VBASE | ✗ 未实证 | ✗ (open, GPU batch vs CPU iterator 不易 fit) |
| Disk-resident segment 友好度 | ✓✓ | ✓ | n/a (GPU memory only) |
| α-augmented streaming-ready | ✗ (open) | ✗ (open) | ✗ (open) |
| **GPU-native build/search** | **不友好（variable degree warp divergence）** | **不友好（同上）** | **CAGRA 设计 first principles for GPU** |

### 选择决策（updated 2026-05-09）

- **CPU + 简单 single-vector TopK + million scale + 静态 + 内存富余** → NSG
- **CPU + single-vector TopK + add-only + million scale** → HNSW
- **CPU + 内存预算严** → IVF + RaBitQ
- **CPU + HCPS + filter heavy** → HNSW + ACORN
- **CPU + multi-column / range / Join + SQL** → HNSW + VBASE
- **CPU + billion-scale + memory** → HNSW + Faiss IVF coarse / RaBitQ + IVF
- **CPU + billion-scale + SSD + single-server** → DiskANN (Vamana) / SPANN
- **CPU + vector DBMS segment + disk-resident** → Starling-HNSW / Vamana / NSG
- **CPU + billion-scale + streaming graph 路径** → FreshDiskANN + FreshVamana
- **CPU + billion-scale + streaming + memory budget 严** → SPFresh + LIRE
- **GPU + dataset 装得进 GPU memory + 静态 (NEW)** → **CAGRA (NVIDIA RAFT / Milvus GPU_CAGRA)**
- **GPU + 需要 single-query 低 latency** → **CAGRA multi-CTA mode**
- **GPU + 需要 large-batch high throughput** → **CAGRA single-CTA mode** 或 Faiss-GPU IVFPQ（按 recall vs memory trade-off）

### CAGRA graph quality 与 NSSG (NSG variant) comparable（NEW）

[per benchmarks/cagra-vs-hnsw-ggnn-ganns.md "Result 7" + ootomo-2023-cagra §V-B Q-C2]

CAGRA paper 把 CAGRA graph 装入 NSSG search 实现（替换 NSSG 自家 graph）→ **recall-throughput 与 NSSG graph 几乎重合**。证明 CAGRA graph quality 不低于 NSSG（NSG 后继）+ build 速度 5-30× faster on GPU。

→ GPU 路径下 graph quality 不损失——只是 algorithm + hardware 同步演进。

### 已知盲区

- **HNSW α-augmented streaming patch** (CPU)：理论可行未做
- **NSG α-augmented streaming patch** (CPU)：同上
- **CAGRA streaming insert/delete** (GPU)：fixed-degree 比 variable-degree 难——open
- **CAGRA + filter / multi-vector / range / Join**：完全 open
- **HNSW vs NSG 在 RaBitQ + IVF 配合下的对比**：相对未深入
- **GPU + multi-segment Milvus 协调 with CAGRA**：当前 Milvus GPU_CAGRA 是 per-segment static；segment-level coordination 未深入

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [systems/cagra.md](../../systems/cagra.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
