---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [benchmarks/faiss-trillion-scale.md, systems/spann.md, systems/diskann.md, concepts/nsg.md, topics/index-selection.md, topics/disk-vs-memory-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 7
---

# Pre-snapshot: giga-scale-sharding

## TL;DR

按当前 wiki 覆盖，工业模板有三条参考路径：(1) **Meta 1.5T Faiss 模式**——分布式 mmap + 极致压缩；(2) **NSG @ Taobao 模式**——内存分片 32 分区；(3) **SPANN @ Bing 模式**——分区 + closure + query-aware dispatch。给定的 16 节点 × 1TB RAM 总预算 16 TB，**够直接走变体 (1) 的简化版**；详细分片层级与 768-d 实测延迟 wiki **未直接覆盖**。

## Answer

### 总数据量与预算估算

万亿向量 × 768-d × float32 = ~3 PB raw；千亿同口径 ~300 TB。16 × 1TB = 16 TB 内存——必须压缩 + 分布式存储。

[per benchmarks/faiss-trillion-scale.md] Meta 1.5T 实测：PCAR72,SQ6 把 144-d 压到 54 字节/向量，总 83 TiB，分布到 20 服务器 mmap。换算到 768-d：PCAR 降到 ~360 + SQ6 ≈ 270 字节/向量，1T → 270 TB；远超 16 TB 内存，必须 SSD-resident。

### 三条工业参考路径

**(1) Faiss IVF + HNSW coarse + 压缩 + 分布式 mmap** [per benchmarks/faiss-trillion-scale.md]
- 10M centroids HNSW 做路由；2000 shards over IDs → 100 lists over centroids
- Meta 实测：单查询 ~1s（20 中央服务器）。**远超 P99 < 50ms 目标**——网络 fetch 倒排 list 是瓶颈。

**(2) NSG @ Taobao 32 partition** [per concepts/nsg.md, queries/index-architecture-global-vs-routed.md]
- 数据切 32 分区，每分区独立 NSG，全内存
- 2B × 12h 索引 × 32 partition；5ms 单查询响应（Taobao 实测）
- **限制**：每分区数据量受单机内存约束。768-d float = 3 KB/vec → 1TB RAM 单机最多 ~3 亿向量；千亿要 ~330 节点（远超 16），万亿要 ~3300 节点

**(3) SPANN HBC + closure + query-aware dispatch** [per systems/spann.md]
- Centroids（~16% N）+ SPTAG 内存索引 + posting list 全精度 SSD
- Bing 几千亿规模生产部署
- 32 节点场景实测：query-aware partitioning 让平均 dispatch 仅 6.3 节点（vs random 32 的 32），省 80.3% IO
- **关键收益**：低 latency budget 下系统性领先 DiskANN（90% recall ~1ms）；适合 read-heavy + 严格 latency

### 16 × 1TB × 128U 私有云的具体落地（基于 wiki 推断）

[per topics/index-selection.md, topics/disk-vs-memory-ann.md] 给定约束下推荐 **SPANN 风格 + IVF + HNSW coarse**：

- 总数据 **千亿** 档：
  - Centroids ~10M（占 ~16% × 1B per node = 160M centroid 候选；缩到 10M）
  - 内存预算：HNSW coarse on centroids ≈ 几十 GB/节点；剩余给 vertex cache + closure replicas
  - SSD：每节点 ~20 TB 装 1/16 数据（千亿 / 16 ≈ 6.25B per 节点 × 270 字节 ≈ 1.7 TB 压缩后；不压缩 ~19 TB 全精度）
  - 分布式 dispatch：query-aware → 平均 6-8 节点参与单查询 [per systems/spann.md §4.3]

- 总数据 **万亿** 档：当前 wiki 实测案例只到 1.5T（Meta），且延迟到秒级；**P99 < 50ms 在 wiki 覆盖范围内无现成方案**

### 周级别全量更新

> [推测，wiki 未覆盖]
> - NSG / Vamana / HNSW 都不支持高效 delete；周级别"全量更新"实际等于 rebuild。
> - 单分区 rebuild 时间：[per benchmarks/faiss-trillion-scale.md] 1.5T 三阶段构建用了数百节点 × 64 核，每周一次成本极高
> - 工业上的常见妥协（wiki 未覆盖）：双索引 + 切换、增量 patch index 叠加、segment-based 模型（Milvus / Pinecone 路线）

### 已知盲区（trigger 后续 ingest）

- **Milvus segment-based 架构**：分布式向量 DB 的事实工业标准，wiki 完全未覆盖
- **Pinecone pod-based 架构**：商业云向量 DB 主流形态
- **768-d 实测延迟**：当前 wiki benchmark 维度上限 144-d（Meta）/ 128-d（SIFT）；现代 768/1024-d 区间 wiki 全未实测
- **周级别全量更新工业实践**：rebuild + 双索引切换、增量 patch、CDC 等工程模式

## Cited Pages

- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
