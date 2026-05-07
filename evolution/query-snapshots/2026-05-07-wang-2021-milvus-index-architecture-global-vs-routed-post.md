---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [queries/index-architecture-global-vs-routed.md, topics/index-selection.md, topics/disk-vs-memory-ann.md, benchmarks/faiss-trillion-scale.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, systems/faiss.md, systems/diskann.md, systems/spann.md, systems/milvus.md, concepts/hnsw.md, concepts/nsg.md, concepts/product-quantization.md]
cited-count: 12
---

# Post-snapshot: index-architecture-global-vs-routed

## TL;DR

工业千亿/万亿主流走 **(c) 层次路由**——四种典型形态：**Meta 1.5T Faiss mmap**（IVF + HNSW coarse + SQ6）、**SPANN @ Bing**（centroids + SSD posting）、**DiskANN**（"(a) 磁盘变体"——单一 graph 但 SSD）、**新增 Milvus segment-based DBMS**——每 segment 1 GB 独立 index + LSM merge + shared-storage 分布式。本次 ingest **填补了 query archive 明确 flag 的"Milvus segment 模型未覆盖"盲区**。

## Answer

### (a) 全局单一索引在千亿+ 的硬约束

- 全内存 graph 1B+ OOM；1.5T × 144d × 4B = 864 TiB raw
- **DiskANN 是 (a) 的"磁盘变体"反例**：单一 Vamana graph + SSD，SIFT1B 单机 64 GB RAM 实测 98.68% recall @ <5ms

### (c) 层次路由的具体形态（updated with Milvus）

**形态 1：Meta 1.5T Faiss** [per benchmarks/faiss-trillion-scale.md]
- 10M centroids HNSW 路由 → IVF 倒排桶 → SQ6 压缩到 54 字节/向量
- 三阶段构建：2000 shards over IDs → 100 lists over centroids → 20 服务器 mmap 83 TiB
- 单查询 ~1 秒

**形态 2：SPANN @ Bing** [per systems/spann.md]
- Centroids 在 DRAM（占 ~16% N），posting list 全精度在 SSD
- HBC + closure clustering + query-aware dynamic pruning
- 1B SIFT @ 90% recall ~1 ms
- 32 partition × bin-packing → 单查询平均 dispatch 仅 6.3 节点

**形态 3 (NEW)：Milvus segment-based DBMS** [per systems/milvus.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md]
- **Segment 是基本单元**（默认 1 GB），每 segment 内 index + 数据同存
- Index 类型 segment 级别可异：IVF_FLAT / IVF_SQ8 / IVF_PQ / HNSW / RNSG
- **LSM tiered merge**：MemTable → flush → 后台合并相近大小 segment
- **Shared-storage 分布式**：单 writer + 多 reader + S3/HDFS；K8s 弹性扩缩
- **Snapshot isolation** 在 segment 层级
- 12 节点 SIFT1B 近线性扩展（IVF_FLAT）

**Faiss 决策树规定** [per topics/index-selection.md Step 2]：
| N | 推荐 |
|---|---|
| 100M – 1B | IVF256k_HNSW + 压缩 |
| > 1B | IVF1M_HNSW + PQ/SQ |

### Trade-off 表（updated with Milvus）

| 维度 | (a) 全局 graph | (c) 路由：Meta/SPANN | (c) 路由：**Milvus segment** |
|---|---|---|---|
| 查询延迟 | < 1B 毫秒级；> 1B OOM；DiskANN 1B @ <5ms | Meta 1.5T ~1s；SPANN 1B @ ~1ms | SIFT10M HNSW ~ms 级；1B+ 论文未拆数字 |
| 构建成本 | 单机线性；> 100M 不现实 | 分布式 cluster job | LSM 异步 flush + tiered merge，**持续低成本** |
| 召回率 | 全精度 ≥95%；DiskANN 98.68% | IVFPQ ~70% / DiskANN re-rank 98% / SPANN >90% | 取 segment 内 index 类型决定 |
| 增量更新 | HNSW add 不 delete；NSG 不增 | IVF add；rebuild 代价高 | **✓ LSM 原生支持持续 insert/delete + tiered merge** |
| 运维 | 单机简单 | 多阶段 + 分布式 mmap + 网络调优 | K8s 自动 + Zookeeper coordinator + S3/HDFS |
| Attribute filter | ✗ | ✗ / 弱（DiskANN 后继 Filtered-DiskANN） | **✓ 五策略** [per topics/attribute-filtering.md] |
| Multi-vector query | ✗ | ✗ | **✓ fusion + iterative** [per topics/multi-vector-queries.md] |

### Milvus segment 模型的关键差异

[per systems/milvus.md, topics/disk-vs-memory-ann.md §"DBMS 层的'内存 + 异步刷盘'模式"]：

不像 (a) 全局或 (c) Meta/SPANN 形态把 index 视作 single global object，Milvus 把 index 切成多 segment 并允许：
- Segment 级别的版本（multi-version snapshot isolation）
- Segment 级别的更新（per-segment merge / rebuild）
- Segment 级别的调度（K8s 把不同 segment 分到不同 reader）
- Segment 级别的 index 选择（同一 collection 不同 segment 可用不同 index）

> **wiki 解读**：Milvus 是 (c) 路由的 **DBMS 化版本**——routing 不仅是 query-time 的 coarse-quantizer 决策，更是 storage-time 的 segment 切分决策。这与 Meta/SPANN 把 routing 仅视为 query optimization 的视角不同。

### 已知盲区（trigger 后续 ingest）

- **Pinecone pod-based 架构**：商业云向量 DB 主流形态，wiki 仍未覆盖；与 Milvus shared-storage 是两条 (c) DBMS 路线
- **DiskANN 磁盘 fan-out 极限**：1B 成功能否外推到 100B+
- **Milvus 2.0+ cloud-native 重写**：1.x segment 模型在 cloud-native 重新分层（log broker + DataNode + QueryNode）后形态可能不同
- **(c) DBMS 模式在万亿规模实测**：Milvus 论文实测到 SIFT1B / 12 节点；万亿规模 DBMS 形态未覆盖
- **多 collection 跨 segment 路由策略**：Milvus 单 collection segment 模型清晰；多 collection 联合查询的路由 wiki 未覆盖

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [systems/faiss.md](../../systems/faiss.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
