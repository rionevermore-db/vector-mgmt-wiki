---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [systems/pase.md, systems/analyticdb-v.md, systems/milvus.md, benchmarks/pase-vs-cube-freddy.md, topics/disk-vs-memory-ann.md]
cited-count: 5
---

# Post-snapshot (yang-2020-pase): giga-scale-sharding

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 不是千亿规模的可选方案**——论文显式承认 PASE billion-scale ✗ + distributed ✗（[wang-2021-milvus Table 1] 印证）。但 PASE 提供"OLTP RDBMS-extended-vector"路径作 **架构对照**——若已有 PG 投资 + transaction-heavy workload，**应用层分片到多 PG 实例**是工程权宜。给定 16 × 1TB 千亿场景：PASE 不适合（million-scale per instance）；仍优先 Milvus / Manu / SPFresh 路径。

## Answer

### PASE 在千亿规模的明确局限（NEW）

[per systems/pase.md "Scale 边界" + wang-2021-milvus Table 1]

PASE 论文承认能力上限：
- **Billion-scale ✗** —— 单 PG 实例 million-scale；多实例需应用层分片
- **Distributed ✗** —— PG 单机；Citus / PolarDB 等分布式 PG 论文未实测
- **Multi-thread ✗** —— PG 单线程查询模型（产线用 multi-process）

→ 给定 16 × 1TB 千亿场景：单 PASE 实例每节点 million-scale ≈ 16M（远小于千亿）；需要应用层分到 ~5000 个 PG 实例——明显不实用。

### 但 PASE 在 transaction-heavy 场景的特殊价值（NEW）

[per systems/pase.md "PG 全套 OLTP 能力 free 复用"]

如果 16 × 1TB 私有云的实际 workload 是：
- **transaction-heavy**（vector + scalar update 频繁、要 ACID 一致性）
- **已有 PG 投资**（schema / 应用层 / 运维已 PG 化）
- **数据规模实际 < billion**（Ant Financial 实际多场景在 million 级）
- **vector 是 secondary signal**（OLTP 主用 SQL + structured data，vector 加性优化）

→ **PASE 是 transaction-heavy 加 vector 的最佳工程权衡**。但与本 query 的"千亿 + read-heavy"约束不匹配。

### 不同 architecture 路径在千亿场景下的适用性（updated）

| 路径 | 千亿可行性 | 备注 |
|---|---|---|
| Library (Faiss) | ✓（trillion 实测）| 用户胶合 |
| 算法系统 (DiskANN/SPANN/SPFresh) | ✓（1B+ 实测） | 单机 + 算法 |
| Vector-first DBMS (Milvus/Pinecone) | ✓（千亿+ production） | **首选** |
| **OLAP-extended (ADBV)** | **✓（13B production）** | OLAP query 友好 |
| **OLTP-extended (PASE)** | **✗（million-scale 上限）** | **不适合千亿** |

### 与之前 ingest 的 ADBV 对比（NEW）

[per systems/pase.md "与 [AnalyticDB-V] 的关系"]

ADBV 与 PASE 是 Alibaba ecosystem **同代但不同 host DB** 的两条非 vector-first 路径：

| | ADBV | PASE |
|---|---|---|
| Host DB | OLAP (AnalyticDB) | OLTP (PostgreSQL) |
| Distributed | ✓ | ✗ |
| Billion+ | ✓ | ✗ |
| Transaction | weak | **PG ACID full** |
| 千亿可行 | **✓** | **✗** |

→ 同 ecosystem 覆盖 OLAP / OLTP 两大场景；千亿规模 ADBV 是答案，PASE 不是。

### 已知盲区

- **PASE 多实例 + 应用层分片在千亿规模实测**：论文未涉及；推断不实用但未量化
- **Distributed PG (Citus / PolarDB) + PASE 实测**：理论可行未实证
- **PASE vs pgvector / pgvecto.rs 在 million-scale 实测**：wiki 未 ingest 后两者
- **2026 SIGMOD 跨 model 整合**：talk 当日 demo

## Cited Pages

- [systems/pase.md](../../systems/pase.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/pase-vs-cube-freddy.md](../../benchmarks/pase-vs-cube-freddy.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
