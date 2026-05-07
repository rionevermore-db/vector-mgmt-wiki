---
title: 索引架构在千亿/万亿规模：全局单一索引 vs 层次路由
type: query
sources: [douze-2024-faiss-library, subramanya-2019-diskann, jegou-2011-pq, malkov-2016-hnsw, fu-2017-nsg]
related: [../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/faiss-trillion-scale.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../systems/faiss.md, ../systems/diskann.md]
created: 2026-05-07
updated: 2026-05-07
---

# 索引架构在千亿/万亿规模：全局单一索引 vs 层次路由

**Date**: 2026-05-07

**Question**:
在千亿/万亿规模下，索引架构应选：
(a) **全局单一索引**——所有向量进同一个索引（如单个 HNSW），无路由层
(c) **层次路由结构**——粗粒度聚类先定位到少数分区，再走那些分区的局部索引（IVF / SPANN / Milvus segment 模型为代表）

两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？

## TL;DR

工业千亿/万亿规模**主流走 (c) 层次路由**，且 (a) 全局 graph 在这一档已被反证不可行——Meta 1.5T 部署给出具体 (c) 形态（IVF + HNSW 10M coarse quantizer + SQ6 + mmap 分布式），全局 graph 在该量级**没有文献支持的成功案例**。

## Answer

### (a) 全局索引在千亿+ 的硬约束

- 全内存 graph (HNSW/NSG) 在 1B+ 直接 OOM——Faiss 1B SIFT 用 IVFPQ 才装下 [per [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)]
- 物理事实：1.5T × 144d × 4B = 864 TiB，不压缩任何单机集群放不下 [per [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)]
- NSG @ Taobao 2B 已经是**多机 32 分区** [per [concepts/nsg.md](../concepts/nsg.md)]——这本身就是某种 (c)，而非真正的全局单一索引

### (c) 层次路由的具体形态

**Faiss 决策树规定** [per [topics/index-selection.md](../topics/index-selection.md) §决策流程 Step 2]：

| N | 推荐 |
|---|---|
| 100M – 1B | `IVF256k_HNSW + 压缩`（HNSW 作 coarse quantizer 充当路由层） |
| > 1B | `IVF1M_HNSW + PQ/SQ`（强制压缩 + 路由） |

**Meta 1.5T 实测** [per [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)]：
- 10M centroids HNSW 做路由 → IVF 倒排桶 → SQ6 压缩到 54 字节/向量
- 三阶段构建：2000 shards over IDs → 100 lists over centroids → 20 服务器 mmap 83 TiB
- 单查询 ~1 秒（中央机器分散到 20 中间服务器后；单中央机器 ~12 秒，瓶颈是 fetch 倒排 list 的网络带宽）

### Trade-off 表（基于 wiki 现有 source）

| 维度 | (a) 全局 graph | (c) 层次路由 |
|---|---|---|
| 查询延迟 | < 1B 毫秒级；> 1B OOM | 1.5T 实测 ~1s |
| 构建成本 | 单机线性；> 100M 不现实 | 分布式 cluster job（数百节点 × 64 核 × 256 GB） |
| 召回率 | 全精度，≥ 95% | 量化失真使上限 ~70%（IVFPQ）；HNSW coarse + SSD 全精度 re-rank（DiskANN 模式）可推到 98% |
| 增量更新 | HNSW 支持 add 不支持 delete；NSG 不支持增量 | IVF 类支持 add；倒排表 rebuild 代价高 |
| 运维 | 单机/分片，简单 | 多阶段 + 分布式 mmap + 网络带宽调优 |

[per [topics/index-selection.md](../topics/index-selection.md), [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)]

### wiki 当前未覆盖的 (c) 重要变体

> [推测，wiki 未覆盖]
> - **SPANN (Chen 2021)**：SSD-resident posting list + 内存 head clusters，与 Faiss 的 IVF + HNSW coarse 路线对立。`disk-vs-memory-ann.md §Open Questions` 已 explicit flag 未 ingest
> - **Milvus segment 模型**、**Pinecone pod-based 架构**：(c) 的工业 DBMS 变体，wiki 无对应 page
> - **DiskANN graph + SSD** [per [systems/diskann.md](../systems/diskann.md)] 严格说不是 (c) 而是另一类——单一 Vamana graph 但下放到 SSD。算 (a) 的"磁盘变体"，从 wiki 看是 (a) 路线在 1B 区间的真正反例（98.68% recall @ <5ms，64GB RAM + SSD）

## Cited Pages

- [topics/index-selection.md](../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)
- [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)
- [concepts/hnsw.md](../concepts/hnsw.md)
- [concepts/nsg.md](../concepts/nsg.md)
- [concepts/product-quantization.md](../concepts/product-quantization.md)
- [systems/faiss.md](../systems/faiss.md)
- [systems/diskann.md](../systems/diskann.md)

## Follow-up Questions

- SPANN 的 SSD-resident posting list 与 Faiss IVF + DiskANN graph 路线的具体性能对比（recall / latency / build cost）？
- Milvus segment 与 Pinecone pod 的边界条件（segment/pod 大小上限、跨 segment 路由策略）？
- DiskANN 的 64 GB RAM + SSD 在 1B 的成功，是否能外推到 100B+？磁盘 fan-out 的极限在哪？
- (c) 在增量写入场景（每天百亿写入）下，posting list 重平衡如何与查询并存？
- 1B+ 规模下 (c) 的召回率天花板：HNSW coarse quantizer + SSD re-rank 能否突破 PQ 的 70% 上限？

---

## 附录：Without-Wiki Baseline（talk 演示对比，非核心）

> ⚠️ **本附录是 2026 部门 talk 的演示对比素材**——LLM 仅凭训练知识回答同一 query 的版本，用于"with vs without wiki" 对照 slide。Talk 结束后可整段删除，不影响 wiki 本体。

### TL;DR (without wiki)

千亿/万亿规模工业主流是 (c) 层次路由。(a) 全局单一索引（单 HNSW）受单机内存、构建时间和图导航跳数三重制约，到 10 亿级就开始撑不住；千亿/万亿只能走 (c)，区别只是路由层和局部索引的具体形态。

### Answer (without wiki)

**(a) 全局索引为何撑不到千亿**：
- HNSW 之类全图算法构建是单机串行的，规模超过 10 亿时构建时间和内存都成问题
- 单一图遍历的 hop 数随 log N 增长，但图越大缓存友好性越差
- 千亿向量即使压缩到 100B 左右也要 10 TB+，单机放不下

**(c) 层次路由的常见形态**：
- 经典 IVF 系：用 k-means 训出 √N 量级 centroids，每个向量分到最近 centroid 形成 posting list，查询先定位 centroids 再精搜——Faiss 处理 1B+ 的主力
- SPANN (Microsoft 2021)：IVF posting list 放 SSD，head clusters 在内存
- DiskANN (Microsoft)：graph + SSD 路线，思路与 IVF 不同
- Milvus：segment-based，每个 segment 内部用 IVF 或 HNSW
- Pinecone：pod-based 架构

**Trade-off 大致对比**：

| 维度 | (a) 全局 graph | (c) 层次路由 |
|---|---|---|
| 查询延迟 | 中小规模毫秒级；超大规模无法做到 | 千亿级通常秒级 |
| 构建成本 | 单机线性，超大规模不可行 | 分布式构建，工程复杂 |
| 召回率 | 全精度，高 | 量化压缩通常有损，需 re-rank |
| 增量更新 | HNSW 支持 add 不支持 delete | IVF 支持 add，倒排重平衡需 rebuild |
| 运维 | 简单 | 多阶段 + 分布式 + 路由调优 |

**为何走 (c)**：单机内存上限 + 构建超线性增长 + 增量更新需求。(c) 几乎是被迫——问题不是"选不选 (c)"，而是"哪种 (c) 变体最适配 workload"。

### with vs without wiki 差异表（talk 用对比素材）

| 维度 | with wiki | without wiki |
|---|---|---|
| 框架结论 | 选 (c) | 选 (c) ✓ 一致 |
| 具体数字 | Meta **1.5T × 144d** = 864 TiB, **54 字节/向量**, **10M centroids HNSW**, **83 TiB**, **20 服务器**, **~1s** 单查询 | 全是泛述，无 anchor |
| 决策阈值 | Faiss 决策树明确 100M–1B 用 `IVF256k_HNSW`，>1B 用 `IVF1M_HNSW` | 模糊"10 亿级开始撑不住" |
| Source 可追溯 | 8 个 page 双向链接到 4 篇论文 + 1 个 benchmark | 凭记忆 list 名字（SPANN/DiskANN/Milvus/Pinecone）但说不出具体规模 |
| 已知盲区 | 明确 flag SPANN/Milvus segment/Pinecone pod **未 ingest** → 直接生成下次 ingest 优先级 | 不知道自己不知道什么 |
| 幻觉风险 | 低（每个数字都 cited） | 中——"√N 量级 centroids"是经验法则，但 N=1.5T 时实测偏离了（10M / √1.5T ≈ 8，低于推荐区间 15-20）。没 wiki 我不会知道这个偏离 |

### Talk 用法

- 开场 promise 段并排展示两版 TL;DR + Trade-off 表
- 主张："专家 vs 业余的差别不在结论方向，而在那些具体数字"
- 这是 wiki 价值最直观的"成果证明"——结构对、数字真，才是专家
