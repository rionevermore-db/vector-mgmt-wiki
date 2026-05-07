---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/proximity-graph.md, concepts/vamana.md, benchmarks/nsg-vs-graph-anns-million.md, benchmarks/hnsw-vs-faiss-200m-sift.md, topics/index-selection.md]
cited-count: 7
---

# Pre-snapshot: hnsw-vs-nsg-selection

## TL;DR

数据能装内存 + 高 precision (≥0.95) → **NSG 系统更优**（QPS 更高、内存约 1/3、构建时间相当或更快）；需要**增量 add** → HNSW（NSG 不支持）；**简单部署 + 工业标准** → HNSW（生态更广）；**SSD 部署** → 两者都不直接支持，应转 Vamana/DiskANN。

## Answer

### 性能对比（million-scale 实测）

[per benchmarks/nsg-vs-graph-anns-million.md] NSG 论文 §4.1 在 SIFT1M / GIST1M / RAND4M / GAUSS5M 上系统对比：

| | NSG | HNSW |
|---|---|---|
| 内存（SIFT1M） | **153 MB** | 451 MB（仅底层）[per concepts/nsg.md] |
| 索引时间（SIFT1M） | 140 + 134s ≈ 274s | 376s |
| Million-scale QPS @ 高 precision (≥0.95) | **优** | 次（差距随 precision 升高扩大） |
| LID 越高（GIST > RAND > SIFT） | NSG 相对优势越大 | — |
| 连通性 SCC（所有 4 数据集） | 1（保证） | 1（保证） |

[per concepts/hnsw.md table] HNSW 在 200M SIFT 上仍是参考实现（HNSWlib）；NSG 在更大规模需多机分片（NSG @ Taobao 32 partition × 2B 数据）。

### 算法机制差异

[per concepts/nsg.md §"与 HNSW 的关键差异"]：

| | NSG | HNSW |
|---|---|---|
| 层数 | **单层** | 多层 |
| Entry point | 单一 Navigating Node（centroid 邻域） | 顶层最高 level 节点 |
| 长程连接 | MRNG 边选择保留远邻 | 顶层显式长程边 |
| 理论基础 | MRNG / MSNET（论文证明 close-log 复杂度） | RNG 近似（经验 O(log N)） |
| 最大出度 | 常数 C_d（理论） | 经验有界 |

### 工程取舍

| 维度 | HNSW 优势 | NSG 优势 |
|---|---|---|
| 增量 add | ✓（HNSW 设计支持，NSG 不支持） | ✗ |
| 增量 delete / update | 都不支持（HNSW Open Q）[per concepts/hnsw.md] | ✗ |
| 内存预算 | 更大（约 NSG 的 3×） | **小** |
| 高 precision QPS | 次 | **优** [per benchmarks/nsg-vs-graph-anns-million.md] |
| 工业生态 | 事实标准（Faiss IndexHNSW、HNSWlib、Milvus、ScaNN-coarse、Pinecone）[per concepts/hnsw.md] | 较新；Faiss IndexNSG、Milvus 已集成 |
| 分布式 | 困难（简单分片） | 困难（Taobao 已用分片） |
| 理论 | 高维度数有界仅经验 | MRNG 给出 close-log 期望复杂度证明 |

### 选型决策树

[per topics/index-selection.md Step 2]：

- **N < 1M + 精度优先 + 一次性构建**：NSG（最快 trade-off，但不支持增量）
- **N < 1M + 增量需求**：HNSW
- **N 1M-100M + 内存富余**：仍 NSG 或 HNSW，差距随 N 缩小
- **N > 100M**：[per topics/index-selection.md] 推荐 IVF + HNSW-as-coarse-quantizer，graph 仅在 coarse 层
- **SSD 部署**：转 [Vamana](../../concepts/vamana.md)（α-controlled graph，diameter 小、SSD 友好）

### Open / 未覆盖

- **多线程吞吐**：[per benchmarks/scann-glove1.2m-mips.md] 单线程对比；并发场景下 graph 吞吐扩展性 wiki 未覆盖
- **新硬件（CXL、HBM）**：wiki 未涵盖
- **NSG 在 MIPS 任务下未验证**：MRNG monotonicity 证明依赖 L2，MIPS 不是 metric [per topics/mips-vs-l2-nn.md]

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/proximity-graph.md](../../concepts/proximity-graph.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [benchmarks/nsg-vs-graph-anns-million.md](../../benchmarks/nsg-vs-graph-anns-million.md)
- [benchmarks/hnsw-vs-faiss-200m-sift.md](../../benchmarks/hnsw-vs-faiss-200m-sift.md)
- [topics/index-selection.md](../../topics/index-selection.md)
