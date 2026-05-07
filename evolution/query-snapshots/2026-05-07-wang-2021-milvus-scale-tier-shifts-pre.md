---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, systems/faiss.md, systems/diskann.md, systems/spann.md, benchmarks/faiss-trillion-scale.md, concepts/nsg.md, concepts/product-quantization.md]
cited-count: 8
---

# Pre-snapshot: scale-tier-shifts

## TL;DR

Wiki 现有 source 给出**三个明确质变点**：(1) **~10M → 1B**：纯 graph 触顶 → 必须 IVF + 量化（HNSW-as-coarse 或 PQ）[per topics/index-selection.md Fig 10]；(2) **1B 单机 RAM 触顶**：内存 → SSD 路线（DiskANN graph + SSD 或 SPANN IVF + SSD）[per topics/disk-vs-memory-ann.md]；(3) **百亿+ → 万亿**：单机 SSD 触顶 → 多机 mmap + 极致压缩（Meta 1.5T 模式）[per benchmarks/faiss-trillion-scale.md]。**百亿与千亿之间的具体阈值**、Milvus / Pinecone 段式架构的"分布式数据库"质变 wiki **未覆盖**。

## Answer

### 三轴跨档质变表（基于 wiki 现有 source）

| 规模档 | 索引选择质变 | 存储介质质变 | 并行能力质变 |
|---|---|---|---|
| **< 10M** | Flat brute force 即可 | DRAM 单机 | 无需 |
| **10M – 1M级 → 1M-100M** | NSG/HNSW (full graph) → IVFflat / IVF+HNSW coarse [per topics/index-selection.md Step 2] | DRAM | 单机多核 |
| **100M – 1B** | IVF+HNSW coarse + 压缩 (PQ/SQ) [per topics/index-selection.md] | DRAM 仍可，但 64 GB+ 工作站 | 单机多核 |
| **1B 跨档质变（最重要的一档）** | **graph 全内存触顶**：HNSW 在 1B SIFT 直接 OOM [per concepts/nsg.md, topics/disk-vs-memory-ann.md]；分支：(a) 全压缩 IVFPQ；(b) DiskANN（Vamana + PQ DRAM + SSD 全精度）；(c) SPANN（centroids DRAM + posting list SSD）；(d) NSG 多机分片 (Taobao 2B 模式) | **DRAM → SSD** 是核心质变；单机 64 GB RAM + NVMe SSD 是新基线 [per systems/diskann.md] | 单机 / 多机均可；NSG @ Taobao 走 32 partition |
| **1B – 100B** | IVF1M_HNSW + PQ/SQ [per topics/index-selection.md Step 2 表]；SPANN 在低 latency 下领先 [per benchmarks/spann-vs-diskann-billion.md] | 必须 SSD（或多机 DRAM 分片） | 必须分布式（多 SSD shard 或多机内存） |
| **100B → 1.5T（Meta 实测）** | IVF + SQ6 + 10M HNSW coarse + 分布式 mmap [per benchmarks/faiss-trillion-scale.md] | mmap 分布式存储（83 TiB / 20 服务器） | **20 中央服务器 + 网络 fan-out 是新瓶颈**（中央机器单查询 ~12s，分散到 20 后 ~1s） |
| **> 1.5T** | wiki **未覆盖** | wiki **未覆盖** | wiki **未覆盖** |

### 三个明确的"质变点"

**质变点 1：~10M → 1M+ 量级时**（[per topics/index-selection.md] Fig 10 决策树阈值）
- Flat brute force → 真正需要索引
- 量级跨度小，争议小

**质变点 2：1B 单机 RAM 触顶**（核心质变，wiki 反复强调）
- HNSW / NSG 全图：1B × 768-d × 4B = ~3 TB raw → 全 graph 不可装
- 三条具体替代路线（DiskANN / SPANN / NSG 多机分片）每条都是论文级工作 [per topics/disk-vs-memory-ann.md]
- **Recall 上限分化**：
  - Faiss IVFPQ 全内存：~62% (IVFOADC plateau)
  - DiskANN graph + SSD 全精度 re-rank：98.68% [per systems/diskann.md]
  - SPANN IVF + SSD 全精度：>90% @ ~1ms [per systems/spann.md]

**质变点 3：百亿+ 走向万亿**
- 单机 SSD 容量触顶（NVMe 单卡 ~10TB 上限；100B × 270 byte ≈ 27 TB 已超）
- 必须分布式 mmap + 极致压缩 [per benchmarks/faiss-trillion-scale.md]
- Meta 1.5T 实测延迟 ~1s（不再 ms 级）—— **延迟语义本身质变**

### 不算质变的（参数微调档）

- 同一 IVF 框架下 K_IVF 从 1024 → 16384 → 1M 仅是参数调
- HNSW 内 M 从 16 → 48 仅是参数调
- 同一 PQ 配置下 m 从 8 → 16 仅是 code length 调

### 已知盲区（trigger 后续 ingest）

- **百亿与千亿之间是否还有质变点**：当前 wiki 把 1B-1.5T 看作连续区间，但工业可能在 50B / 100B / 500B 各有微观质变（如 SSD partition 数、network fan-out 极限）
- **Milvus / Pinecone 段式架构的"分布式数据库"质变**：当向量索引从"工程艺术品"变成"DBMS object" 时（segment 创建/回收/合并、metadata 索引、tenant 隔离），是 wiki 未覆盖的另一种"质变"
- **Read-heavy vs write-heavy 路线分化**：wiki source 全是 read-heavy benchmark；高吞吐写入场景的质变点未覆盖
- **Embedding 升级触发的质变**：模型版本切换是另一种"事件型质变"，[query: embedding-update-handling] 完全未覆盖

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [systems/faiss.md](../../systems/faiss.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
