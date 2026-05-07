---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/proximity-graph.md, concepts/vamana.md, benchmarks/nsg-vs-graph-anns-million.md, benchmarks/hnsw-vs-faiss-200m-sift.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, topics/index-selection.md, systems/milvus.md, systems/faiss.md]
cited-count: 10
---

# Post-snapshot: hnsw-vs-nsg-selection

## TL;DR

数据能装内存 + 高 precision (≥0.95) → **NSG 系统更优**；需要**增量 add** → HNSW（NSG 不支持）；**简单部署 + 工业标准** → HNSW（生态最广，[Milvus](../../systems/milvus.md) graph-based 索引列表里 HNSW 是默认）；**SSD 部署** → 转 Vamana/DiskANN。**新增**：Milvus 把 HNSW 与"RNSG"并列作为两个 graph 选项 [wang-2021-milvus §2.2]，但 RNSG 引用 ref [20] = NSG 而非 ref [61] = Rand-NSG，**身份歧义**——是 NSG 变体还是 Vamana 实现，论文未澄清。

## Answer

### 性能对比（million-scale 实测）

[per benchmarks/nsg-vs-graph-anns-million.md] NSG 论文在 SIFT1M / GIST1M / RAND4M / GAUSS5M 上系统对比：

| | NSG | HNSW |
|---|---|---|
| 内存（SIFT1M） | **153 MB** | 451 MB（仅底层）[per concepts/nsg.md] |
| 索引时间（SIFT1M） | 140 + 134s ≈ 274s | 376s |
| Million-scale QPS @ 高 precision (≥0.95) | **优** | 次（差距随 precision 升高扩大） |
| 连通性 SCC | 1（保证） | 1（保证） |

[per benchmarks/milvus-vs-prior-sift10m-deep10m.md] Milvus 工业实测（HNSW 变体）：
- SIFT10M / Deep10M 上 HNSW throughput >15000-22000 q/s @ recall ≥ 0.95
- HNSW 比 IVF_FLAT 在高 recall 区间更快、更省内存
- Milvus_HNSW 比 Vearch / 商业 A/C 快 7-73×

### 算法机制差异

[per concepts/nsg.md §"与 HNSW 的关键差异"]：

| | NSG | HNSW |
|---|---|---|
| 层数 | **单层** | 多层 |
| Entry point | 单一 Navigating Node（centroid 邻域） | 顶层最高 level 节点 |
| 长程连接 | MRNG 边选择保留远邻 | 顶层显式长程边 |
| 理论基础 | MRNG / MSNET（论文证明 close-log 复杂度） | RNG 近似（经验 O(log N)） |
| 最大出度 | 常数 C_d（理论） | 经验有界 |

### 工业生态对比（**新增 Milvus 数据**）

| 维度 | HNSW | NSG |
|---|---|---|
| 增量 add | ✓ | ✗ |
| 增量 delete / update | 都不支持 | ✗ |
| 内存预算 | 更大（约 NSG 的 3×） | **小** |
| 高 precision QPS（million-scale） | 次 | **优** |
| Faiss `IndexHNSW` / `IndexNSG` | ✓ | ✓ |
| **Milvus 集成** [wang-2021-milvus §2.2] | ✓ **graph-based 索引默认** | ✓ 命名为 "RNSG"，**身份歧义** |
| HNSWlib 生态 | **事实标准**，被 Faiss / Milvus / nmslib / Pinecone 等引用 | 弱（仅 ZJULearning/nsg） |

### Milvus 的 HNSW vs RNSG 选择 [per systems/milvus.md]

[wang-2021-milvus §2.2] 把 graph 索引列为 "HNSW + RNSG"，但 RNSG 引用 ref [20] = Fu 2017 = NSG（不是 Subramanya 2019 ref [61] = Rand-NSG/Vamana）。**正文 ref vs 命名矛盾，论文未澄清**——是 NSG 变体还是 Vamana 工程实现需查 Milvus 源码。

> **wiki 解读**：RNSG 命名歧义意味着 Milvus 用户在选 graph 索引时其实在选什么不清楚。HNSW 是无歧义的工业标准；选 HNSW 可避免命名/语义混淆。

### 选型决策树（updated）

[per topics/index-selection.md]：

- **N < 1M + 精度优先 + 一次性构建**：NSG（最快 trade-off，但不支持增量）
- **N < 1M + 增量需求**：HNSW
- **N 1M-100M + 内存富余**：HNSW（生态 + Milvus 默认）；NSG 在高 precision 下边际优势
- **N > 100M + 单机内存**：HNSW-as-coarse-quantizer（IVF + HNSW + 压缩）
- **N 1B+ + 单机 SSD**：转 [Vamana](../../concepts/vamana.md)（DiskANN）；HNSW/NSG 都 SSD 不友好
- **DBMS 场景**（动态数据 + 分布式 + filter）：[Milvus](../../systems/milvus.md) HNSW 是默认 graph 选项

### Open / 未覆盖

- **多线程吞吐**：[per benchmarks] 多数对比单线程；Milvus 实测多核已含 cache-aware 优化但绝对数字仅在 SIFT10M 量级
- **新硬件（CXL / HBM / FPGA）**：wiki 未涵盖；Milvus §9 提到 FPGA IVF_PQ 但 not graph
- **NSG 在 MIPS 任务下未验证**：[per topics/mips-vs-l2-nn.md] MRNG monotonicity 证明依赖 L2
- **RNSG 真实身份**：source 未澄清

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/proximity-graph.md](../../concepts/proximity-graph.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [benchmarks/nsg-vs-graph-anns-million.md](../../benchmarks/nsg-vs-graph-anns-million.md)
- [benchmarks/hnsw-vs-faiss-200m-sift.md](../../benchmarks/hnsw-vs-faiss-200m-sift.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/faiss.md](../../systems/faiss.md)
