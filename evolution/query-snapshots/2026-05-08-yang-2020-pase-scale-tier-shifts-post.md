---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md, systems/pase.md, systems/analyticdb-v.md, benchmarks/pase-vs-cube-freddy.md]
cited-count: 6
---

# Post-snapshot (yang-2020-pase): scale-tier-shifts

## TL;DR (delta from wei-2020-analyticdb-v post)

**第 8 质变（DB-extended 路径）显式分裂为子维度：OLAP-extended (ADBV) vs OLTP-extended (PASE)**。两者都是非 vector-first，但 host DB 类型决定能力上限：ADBV scale 到 13B production；PASE 仅 million-scale per instance。这是 wiki 第一次见**架构起点的 RDBMS-类型敏感性**——同样"加 vector 到现有 DB"路径，OLAP host 与 OLTP host 行为差异巨大。

## Answer

### 第 8 质变维度的子分裂（NEW）

[per topics/index-selection.md "DB-extended-vector 路径中的 RDBMS 类型分化"]

之前：
- **质变 8**：vector-first DBMS → OLAP-extended-vector

PASE ingest 后：
- **质变 8a**：vector-first → OLAP-extended-vector (ADBV)
- **质变 8b**：vector-first → OLTP-extended-vector (PASE)
- **质变 8c**（推断未实证）：vector-first → key-value-extended (Redis Vector?, wiki 未 ingest)

| 子维度 | 代表 | Scale 上限 | Distributed | Host DB workload 适配 |
|---|---|---|---|---|
| 8a: OLAP-extended | ADBV | **13B production** | ✓ | OLAP / read-heavy |
| 8b: OLTP-extended | **PASE** | **million per instance** | ✗ | **OLTP / transaction-heavy** |
| 8c: KV-extended | wiki 未 ingest | — | — | session-store / cache |

→ "加 vector 到现有 DB" 这条路径**不是单一选择**——具体哪种 DB host 决定了一切。

### Scale 8a vs 8b 的根本差异（NEW）

[per systems/pase.md, systems/analyticdb-v.md]

| | OLAP-extended (ADBV) | OLTP-extended (PASE) |
|---|---|---|
| Host DB 设计目标 | 大数据 read 分析 | 小数据频繁更新 |
| 内存模型 | shared-nothing MPP | shared-buffer 单实例 |
| Storage | 分布式（Pangu） | PG 单机 (+ 多实例 + WAL replica) |
| 单查询延迟 | sub-second | ms 级 |
| Write throughput | bulk insert | 每 transaction |
| Vector scale 上限 | 13B production | **million per instance** |
| Distributed 原生 | ✓ | ✗ |

→ "加 vector"在 OLAP DB 与 OLTP DB 上**得到完全不同的 scale 上限**。这是质变 8 的关键子分化。

### 八+1 个质变点（updated）

| # | 质变点 | 维度 | PASE ingest 影响 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | n/a |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | n/a |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | n/a |
| 4 | library → DBMS（cloud-native） | 工程形态 | n/a |
| 5 | out-of-place rebuild → in-place | update strategy | n/a |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | n/a |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | n/a |
| 8 | vector-first DBMS → DB-extended | 架构起点 | **细分为 8a (OLAP) vs 8b (OLTP)** |
| **8a** | OLAP-extended (ADBV) | 子选择 | scale 上限 13B |
| **8b（NEW）** | **OLTP-extended (PASE)** | **子选择** | **scale 上限 million per instance** |

### 与之前 ingest 的演进

| | wei-2020-analyticdb-v post | **yang-2020-pase post (NEW)** |
|---|---|---|
| 质变维度 | 8（架构起点首次显式） | **8 但分化 8a/8b** |
| DB-extended 路径数 | 1 (ADBV) | **2 (ADBV + PASE)** |
| Host DB 敏感性 | 隐含 | **显式：OLAP vs OLTP 决定能力上限** |
| PASE billion-scale 实测 | n/a | **明确不可达**（论文 + Table 1） |

### 不算质变（参数微调）

- PASE shared_buffers 4GB vs 16GB
- PASE IVFFlat cc=100 vs cc=1000
- PASE HNSW bnn=16 efb=80 vs efb=200

### 已知盲区

- **8c key-value-extended**：Redis Vector / Aerospike Vector 等 wiki 未 ingest
- **Distributed PG (Citus / PolarDB) + PASE 在 billion-scale 实测**：未实证
- **8a 与 8b 的中间形态**：HTAP DBMS（如 TiDB / SingleStore）的 vector 集成 wiki 未涉及
- **2026 SIGMOD 跨 model 整合**：embedding lifecycle 是另一种"事件型质变"

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [benchmarks/pase-vs-cube-freddy.md](../../benchmarks/pase-vs-cube-freddy.md)
