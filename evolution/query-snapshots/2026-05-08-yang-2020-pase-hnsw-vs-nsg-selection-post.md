---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/pase.md, benchmarks/pase-vs-cube-freddy.md]
cited-count: 4
---

# Post-snapshot (yang-2020-pase): hnsw-vs-nsg-selection

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 选 HNSW 但承认 build 慢极**（SIFT 1M HNSW build 4942s vs IVFFlat 195s = **25× 慢**；GIST 1M 56× 慢）。这与 Faiss / Milvus 内的 HNSW 差距比明显大——根因是 **PG 内核单线程约束 + 8KB page-aligned constraint**。HNSW 在 PASE 中的实测延伸"HNSW 在系统约束下表现取决于 host system"的认知。**PASE 不集成 NSG**——单线 graph 选 HNSW。

## Answer

### PASE HNSW build 慢极的工程含义（NEW）

[per benchmarks/pase-vs-cube-freddy.md "Build vs Storage"]

PASE §4.3 实测：

| Algorithm | SIFT 1M Build (s) | GIST 1M Build (s) |
|---|---|---|
| IVFFlat (cc=100, sr=0.01) | 35 | 113 |
| IVFFlat (cc=1000, sr=0.01) | 195 | 372 |
| HNSW (bnn=16, efb=80) | 2013 | 9962 |
| HNSW (bnn=16, efb=200) | 4942 | 20875 |

→ HNSW build 比 IVFFlat **15-100× 慢**——比其他 wiki 系统中的差距大很多。

**根因推测** [per systems/pase.md "Open Q"]：
- PG 内核单线程查询模型——HNSW graph construction 不能并行
- 8 KB page-aligned 约束——HNSW neighbor 边界对齐成本
- PG IndexAmRoutine 接口约束限制 batch insertion

→ 在 wiki 已有 HNSW 实证中：
- Faiss / nmslib HNSW build 单 1M 数据 ~10-30s（多线程）
- Milvus / Manu HNSW 类似 Faiss 量级
- **PASE HNSW build 1M 数据 ~5000s——慢 100×+**

→ **HNSW 在系统约束下表现差异巨大**——不是算法问题，是 host system 问题。

### PASE 选 HNSW 但不集成 NSG（NEW）

[per systems/pase.md "ANN 算法 in PASE"]

PASE Table 选了 **IVFFlat + HNSW** 两实现：
- 不选 NSG：原因 paper 未明示，推测因 NSG 不增量（与 PG transaction 增删改不兼容）
- 不选 SCANN：MIPS 不是金融场景主用
- 不选 DiskANN：单 PG 实例不需要 SSD 路线

→ **PASE 与 ADBV / Manu 一样不集成 NSG**——印证 wiki "graph-based 系统大多放弃 NSG 因不增量"的观察。

### 选型决策树（updated）

| 场景 | 推荐 | 备注 |
|---|---|---|
| 静态 + million-scale + 高 precision | NSG（论文实测最强）| 无增量 |
| 静态 / 增量 add + Milvus / Manu | HNSW | 实测最优 |
| **PG OLTP 内增量 + transaction** | **PASE HNSW**（接受慢 build）或 **PASE IVFFlat**（更快 build）| PASE 内 HNSW build 慢 25-56× |
| Billion-scale + SSD | DiskANN / Manu SSD-aware | NSG / HNSW 都不适合 |

### 与之前 ingest 的演进

| | wei-2020-analyticdb-v post | **yang-2020-pase post (NEW)** |
|---|---|---|
| HNSW 在系统中的表现 | ADBV 用 HNSW 作 streaming layer | **PASE HNSW build 慢极（系统约束）** |
| Graph 在不同 host DB 下的差异 | OLAP 与 vector-first 同性能 | **OLTP RDBMS (PG) 显著退化** |
| NSG 集成情况 | ADBV 不集成 | **PASE 也不集成** |

### Open / 未覆盖

- **PASE HNSW vs Faiss HNSW 直接对比**：相同硬件下差距未实测
- **PG 多线程构建 HNSW 的可行性**：parallel CREATE INDEX 在 PG 11+ 支持（但 PASE 论文为 PG 11 时代）；后续 PG 版本是否改善 HNSW build？
- **pgvector 内 HNSW 与 PASE HNSW 对比**：pgvector wiki 未 ingest

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/pase.md](../../systems/pase.md)
- [benchmarks/pase-vs-cube-freddy.md](../../benchmarks/pase-vs-cube-freddy.md)
