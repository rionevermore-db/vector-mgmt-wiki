---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [systems/diskann.md, systems/spann.md, concepts/vamana.md, concepts/product-quantization.md, topics/disk-vs-memory-ann.md, benchmarks/diskann-sift1b.md, benchmarks/spann-vs-diskann-billion.md, benchmarks/faiss-trillion-scale.md]
cited-count: 8
---

# Pre-snapshot: memory-vs-disk-large-scale

## TL;DR

Wiki 现有 **三条已 ingest 的 SSD 路线 + 一条 mmap 路线**：(1) **DiskANN** = Vamana graph + PQ DRAM 导航 + SSD 全精度 re-rank（Microsoft 2019）；(2) **SPANN** = IVF + 全精度 SSD posting list + closure（Microsoft 2021）；(3) **Faiss IVFPQ** + 多机 mmap（Meta trillion-scale）；(4) **NSG @ Taobao** 多机内存分片（非 SSD 路线）。SPANN 在低 latency budget 下系统性领先 DiskANN（90% recall ~1ms vs ~3-4ms），但 DiskANN 在高 latency budget 下追平。

## Answer

### 路线对比表 [per topics/disk-vs-memory-ann.md]

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 代表 |
|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | HNSW / NSG |
| 量化压缩 + 全内存 | DRAM | ~62% (IVFOADC plateau) | 数十 GB | Faiss IVFPQ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | NSG @ Taobao 32-partition |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | Faiss-GPU |
| **磁盘 + 量化导航 + SSD re-rank** | DRAM (PQ) + SSD | **98.68%** | **64 GB** | **DiskANN** [per systems/diskann.md] |
| **磁盘 + IVF + 全精度 posting list** | DRAM (centroids) + SSD | **>90% @ ~1 ms** | ~32 GB | **SPANN** [per systems/spann.md] |
| 分布式 mmap + 极致压缩 | mmap 分布式存储 | — | 20 服务器 | Meta 1.5T Faiss [per benchmarks/faiss-trillion-scale.md] |

### DiskANN 核心思路 [per systems/diskann.md, concepts/vamana.md]

- **Vamana graph + α-controlled** → 小 diameter（hop 数比 HNSW / NSG 少 2-3×）
- **PQ codes 在 DRAM**（32 byte/vec）做导航；**全精度向量 + graph edges 在 SSD**（每节点 4 KB 扇区共享）
- **Beam search W=4-8** 批量 SSD I/O，每跳读 W 个邻居
- **全精度 re-rank**："读邻居顺手就拿到全精度坐标"——免费突破 PQ 失真天花板
- 1B SIFT 单机 64 GB RAM，1-recall@1 = 98.68% @ <5 ms，>5000 QPS

### SPANN 核心思路 [per systems/spann.md]

- **不用 PQ，全程全精度**——内存只放 centroids（占 ~16% N），SSD 放完整向量
- **Hierarchical Balanced Clustering (HBC)** 切到 12 KB byte / 48 KB float posting list 上限
- **Closure clustering**：边界向量复制到多个最近簇（最多 8 replicas），RNG rule 避免冗余
- **Query-aware dynamic pruning**：ε₂ 阈值控制扫几个 list（不同 query 难度差异大）
- **SPTAG 内存索引** over centroids 做亚毫秒级 nearest-centroid 查询
- 1B SIFT / DEEP / SPACEV 上 90% recall @ ~1 ms，单机 ~32 GB RAM
- Bing 几千亿规模生产部署

### DiskANN vs SPANN 关键差异 [per benchmarks/spann-vs-diskann-billion.md]

| | DiskANN | SPANN |
|---|---|---|
| 算法路线 | Graph (Vamana) + SSD | Inverted file + SSD |
| 是否用 PQ | 是（导航） | 否（全精度） |
| SSD 访问模式 | 多次小读（每跳一次） | **少量大读**（K 个 posting list） |
| 90% recall 延迟 | ~3-4 ms | **~1 ms** |
| 公平 benchmark | 多 latency budget 下混合胜率 | 低 latency budget 下系统性领先 |

**为什么 SPANN 能不用 PQ**：[per topics/disk-vs-memory-ann.md] inverted file 是 *block-sequential*（一次读一个 list），SSD 顺序读带宽足够。Graph 是 *random-pointer-chasing*，每次读小 → IOPS 触顶 → 必须用 PQ 减少候选。

### Trillion-scale 跨档：mmap + 极致压缩

[per benchmarks/faiss-trillion-scale.md] Meta 1.5T × 144-d：
- PCAR72,SQ6 = 54 字节/向量 → 83 TiB 总索引
- HNSW 10M coarse quantizer
- 三阶段构建：2000 shards over IDs → 100 lists over centroids → 20 服务器 mmap
- 单查询 ~1s，瓶颈在网络 fetch 倒排 list

### 已知盲区

- **Milvus segment-based 架构**：分布式向量 DB 标准做法，wiki 完全未覆盖
- **Pinecone pod-based 架构**：商业云向量 DB 主流形态
- **CXL / 持久内存中间层**：DRAM 与 SSD 之间新存储层，未覆盖
- **网络存储下的 ANN**：所有 disk-resident 分析假设本地 NVMe；远程块设备 / 对象存储未覆盖

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/diskann-sift1b.md](../../benchmarks/diskann-sift1b.md)
- [benchmarks/spann-vs-diskann-billion.md](../../benchmarks/spann-vs-diskann-billion.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
