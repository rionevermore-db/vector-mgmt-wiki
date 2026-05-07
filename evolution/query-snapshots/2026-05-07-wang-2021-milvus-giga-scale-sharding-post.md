---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [systems/milvus.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, benchmarks/faiss-trillion-scale.md, systems/spann.md, systems/diskann.md, systems/faiss.md, concepts/nsg.md, topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/attribute-filtering.md, queries/index-architecture-global-vs-routed.md]
cited-count: 11
---

# Post-snapshot: giga-scale-sharding

## TL;DR

工业模板**四条参考路径**（新增 Milvus DBMS 路线）：(1) **Meta 1.5T Faiss** mmap + SQ6；(2) **NSG @ Taobao** 多机内存分片；(3) **SPANN @ Bing** centroids + SSD posting list；(4) **Milvus shared-storage 分布式 DBMS** 是唯一同时具备 dynamic data + distributed 的方案——周级全量更新场景的天然适配。给定 16 × 1TB 总预算 16 TB，**对千亿 + 768-d 推荐 Milvus 分布式部署 + IVF_HNSW + SQ8 + 按高频 filter 属性预 partition**；万亿仍超 wiki 已实测案例（Meta 1.5T 也是秒级延迟，本场景 P99 < 50ms 不可达）。

## Answer

### 总数据量与预算估算

千亿 × 768-d × float32 = ~300 TB raw；万亿 ~3 PB。16 × 1TB = 16 TB 内存——必须压缩 + 分布式存储。

[per benchmarks/faiss-trillion-scale.md] Meta 1.5T 实测：PCAR + SQ6 把 144-d 压到 54 字节/向量 → 83 TiB 总。换算 768-d：PCAR 降到 ~360 + SQ6 ≈ 270 字节/向量，1T → 270 TB。即使 16 节点合计 16 TB DRAM + 配套 SSD（~20 TB/节点 = 320 TB total），千亿压缩后能装下，万亿不行。

### 四条工业参考路径

**(1) Faiss IVF + HNSW coarse + 压缩 + 分布式 mmap** [per benchmarks/faiss-trillion-scale.md]
- Meta 实测 1.5T：单查询 ~1s（20 中央服务器 fan-out）。**远超 P99 < 50ms 目标**

**(2) NSG @ Taobao 32 partition** [per concepts/nsg.md]
- 768-d float = 3 KB/vec → 1TB RAM 单机最多 ~3 亿向量
- 千亿要 ~330 节点（远超 16）；不适用

**(3) SPANN HBC + closure + query-aware dispatch** [per systems/spann.md]
- 32 节点场景 query-aware partitioning 让平均 dispatch 仅 6.3 节点（vs random 32）
- Bing 几千亿规模生产
- **低 latency budget 下系统性领先 DiskANN（90% recall ~1ms）**——直接命中 P99 < 50ms 要求

**(4) Milvus DBMS（NEW，本次 ingest）** [per systems/milvus.md]
- **唯一同时支持 dynamic data + distributed + GPU + attribute filtering + multi-vector** 的工业系统 [per wang-2021-milvus Table 1]
- **Shared-storage 架构**（Snowflake / Aurora 风格）：单 writer + 多 reader + S3/HDFS 共享存储 [per systems/milvus.md §"分布式架构"]
- **Segment-based 调度**：默认 1 GB/segment，每 segment 独立 index + LSM tiered merge
- 12 节点 scaling 实验（ecs.g6e.13xlarge，52 vCPU 192 GB）证明 SIFT1B 近线性扩展 [per benchmarks/milvus-vs-prior-sift10m-deep10m.md Fig 10b]
- **Cache-aware + AVX512 + SQ8H** 工程优化让 IVF 类索引比 vanilla Faiss 快 2.7×/1.5×/系统性

### 16 × 1TB × 128U 私有云的具体落地（基于 wiki 推断）

[per systems/milvus.md, topics/index-selection.md] 给定约束下推荐 **Milvus 分布式部署**：

- **总数据 千亿 档**：
  - 16 节点拆 segment：每节点 ~6.25B / 16 segment-per-node = ~390M vec/segment（若 segment 大小 1 GB 经典值，等价 768-d 下每 segment 仅 ~340k vec——**segment 设大点至 16 GB / segment 更现实**）
  - 索引：IVF_HNSW + SQ8（每 vec 768 byte → 12 TB 千亿，分布到 16 节点 SSD ~20TB/节点 装得下）
  - **HNSW coarse quantizer 路由层**避免 brute-force coarse [per topics/index-selection.md Step 2]
  - **Attribute filtering**：按高频 filter 属性（如 region / time 范围）预 partition（strategy E）[per topics/attribute-filtering.md] —— ~1M vec/分区经验值，千亿 → ~10万分区
  - **Read-heavy + P99 < 50ms**：[per benchmarks/milvus-vs-prior-sift10m-deep10m.md] Milvus 在 SIFT10M HNSW 上 throughput >15000 q/s @ recall ≥ 0.95；千亿规模 latency 论文未实测，但 SPANN 路线 ~1ms 暗示 50ms 可达

- **总数据 万亿 档**：当前 wiki 实测案例只到 1.5T（Meta，~1s 延迟）；**P99 < 50ms 在 wiki 覆盖范围内无现成方案**。Milvus 论文 §9 提"FPGA 加速 IVF_PQ"与 "cloud-native 重写"（Milvus 2.0+），但本论文未实测

### 周级别全量更新

[per systems/milvus.md §"动态数据"] **Milvus LSM segment 模型直接支持**：
- 写入到 MemTable → flush 为 immutable segment → tiered merge
- **out-of-place delete**（marker + merge 时清理）
- Snapshot isolation 让 query 不被并发写入阻塞

周级别全量更新等价于"持续 insert + 旧数据 delete"——Milvus 的 LSM 是该场景的**直接适配**，不像 NSG/Vamana/HNSW 必须 rebuild。

> **wiki 解读**：周级全量更新这道题在 wiki ingest Milvus 之前**几乎无解**——其他四条系统都假设静态数据。Milvus 是 wiki 现有 source 中**唯一原生支持持续更新**的系统级方案。

### 已知盲区（trigger 后续 ingest）

- **Milvus 2.0+ cloud-native 重写**：本论文是 1.x 架构；2.0+ 重新分层（log broker + DataNode + QueryNode）尚未 ingest
- **Pinecone pod-based 架构**：商业云向量 DB 主流形态，wiki 仍未覆盖
- **768-d 实测延迟**：当前 wiki benchmark 维度上限仍在 144/128-d；现代 768/1024-d 区间未实测
- **多属性联合 partition**：Milvus 论文 §4.1 单属性 partition；多属性是 NP-hard，未解 [per topics/attribute-filtering.md Open Q]
- **Embedding 升级触发的 schema migration**：[per query: embedding-update-handling] 完全未覆盖
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**（talk 当日 demo 主题）

## Cited Pages

- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/faiss.md](../../systems/faiss.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
