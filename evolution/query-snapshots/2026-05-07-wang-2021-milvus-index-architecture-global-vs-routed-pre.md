---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [queries/index-architecture-global-vs-routed.md, topics/index-selection.md, topics/disk-vs-memory-ann.md, benchmarks/faiss-trillion-scale.md, systems/faiss.md, systems/diskann.md, systems/spann.md, concepts/hnsw.md, concepts/nsg.md, concepts/product-quantization.md]
cited-count: 10
---

# Pre-snapshot: index-architecture-global-vs-routed

## TL;DR

工业千亿/万亿规模主流走 **(c) 层次路由**——Meta 1.5T 实测形态：IVF + HNSW 10M coarse + SQ6 + 分布式 mmap。(a) 全局 graph 在 1B+ 已被反证不可行（HNSW 全图直接 OOM）。**DiskANN 是 (a) 的"磁盘变体"**——单一 Vamana graph 但下放 SSD，1B 实测 98.68% recall @ <5ms。SPANN 是 (c) 的另一形态——IVF + SSD 全精度 posting list，1B @ ~1ms。当前 wiki 已有 query archive [queries/index-architecture-global-vs-routed.md]，本 snapshot 是其复述。

## Answer

### (a) 全局单一索引在千亿+ 的硬约束 [per queries/index-architecture-global-vs-routed.md]

- 全内存 graph (HNSW/NSG) 在 1B+ 直接 OOM [per topics/disk-vs-memory-ann.md]
- 物理事实：1.5T × 144d × 4B = 864 TiB raw，不压缩任何单机集群放不下
- NSG @ Taobao 2B 已经是多机 32 分区——本身就是某种 (c)，而非真正全局单一索引

**(a) 的"磁盘变体"反例**：[per systems/diskann.md] DiskANN 是单一 Vamana graph，但下放到 SSD。SIFT1B 单机 64 GB RAM + NVMe，1-recall@1 = 98.68% @ <5ms。严格说不算 (c)，是 (a) 路线在 1B 区间的真正反例。

### (c) 层次路由的具体形态

**Meta 1.5T 实测** [per benchmarks/faiss-trillion-scale.md]：
- 10M centroids HNSW 做路由 → IVF 倒排桶 → SQ6 压缩到 54 字节/向量
- 三阶段构建：2000 shards over IDs → 100 lists over centroids → 20 服务器 mmap 83 TiB
- 单查询 ~1 秒（中央机器分散到 20 中间服务器后；单中央机器 ~12 秒）

**SPANN 形态** [per systems/spann.md]：
- (c) 的另一变体：centroids 在 DRAM（占 ~16% N），posting list 全精度在 SSD
- HBC + closure clustering + query-aware dynamic pruning + RNG rule
- 1B SIFT @ 90% recall ~1 ms
- 32 partition × bin-packing 让单查询平均 dispatch 仅 6.3 节点（vs random 32）

**Faiss 决策树规定** [per topics/index-selection.md Step 2]：
| N | 推荐 |
|---|---|
| 100M – 1B | IVF256k_HNSW + 压缩（HNSW 作 coarse 充当路由层） |
| > 1B | IVF1M_HNSW + PQ/SQ（强制压缩 + 路由） |

### Trade-off 表 [per queries/index-architecture-global-vs-routed.md]

| 维度 | (a) 全局 graph | (c) 层次路由 |
|---|---|---|
| 查询延迟 | < 1B 毫秒级；> 1B OOM；DiskANN 磁盘变体 1B @ <5ms | 1.5T 实测 ~1s；SPANN 1B @ ~1ms |
| 构建成本 | 单机线性；> 100M 不现实 | 分布式 cluster job（数百节点 × 64 核 × 256 GB） |
| 召回率 | 全精度 ≥95% | 量化失真使上限 ~70% (IVFPQ)；SSD 全精度 re-rank 可推到 98% |
| 增量更新 | HNSW 支持 add 不支持 delete；NSG 不支持 | IVF 类支持 add；倒排表 rebuild 代价高 |
| 运维 | 单机/分片简单 | 多阶段 + 分布式 mmap + 网络带宽调优 |

### 已知盲区（trigger 后续 ingest）

- **Milvus segment-based 架构**：(c) 的"工业 DBMS 变体"，wiki 完全未覆盖；Wang et al. 2021 SIGMOD 是该路线的奠基论文，本次正在 ingest
- **Pinecone pod-based 架构**：商业云向量 DB 主流形态
- **DiskANN 磁盘 fan-out 极限**：1B 成功能否外推到 100B+
- **(c) 在增量写入场景的可行性**：每天百亿写入下，posting list 重平衡如何与查询并存

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [systems/faiss.md](../../systems/faiss.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
