---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [systems/pase.md, systems/analyticdb-v.md, topics/disk-vs-memory-ann.md, benchmarks/pase-vs-cube-freddy.md]
cited-count: 4
---

# Post-snapshot (yang-2020-pase): memory-vs-disk-large-scale

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 不在百亿+ 范围内**——论文实测仅到 SIFT1M / GIST1M（[wang-2021-milvus Table 1] PASE Billion-Scale ✗）。但 PASE 提供 **OLTP RDBMS 路径（PG 8KB page + buffer + WAL + storage）作为 memory/disk 第 8 条架构**——已 ingest 的所有路径都不直接覆盖此模式。生产中 PASE 用应用层分片到多 PG 实例处理 ~billions（Ant Financial 图像版权检测）但论文未深入分片细节。

## Answer

### 路线对比表完整版（updated with PASE）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机/集群预算 | Update 模型 |
|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB peak | streamingMerge |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB peak | 周期 rebuild |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ |
| DBMS LSM segment / cloud-native | DRAM + S3 | — | ~64 GB/节点 | LSM merge |
| 磁盘 + IVF + 全精度 + In-place 增量 | DRAM + raw NVMe | >0.86 @ ~5 ms | 持续 10 GB | LIRE in-place |
| SaaS + slab + adaptive indexing | Memtable + cache + slab on object storage | docs 未公开 | docs 未公开 | slab merge |
| DBMS + SSD-aware hierarchical k-means + LSH replication | DRAM (centers) + SSD (4KB block + 4-8× replication) | NeurIPS 2021 winner | docs 未明示 | DBMS 全套 |
| OLAP DB + Pangu 分布式存储 + VGPQ batching | DRAM (HNSW streaming) + Pangu (VGPQ batching, distributed) | 13B records / 30 TB production | 70 节点 Alibaba Cloud | Lambda async merge |
| **OLTP RDBMS (PostgreSQL) + 8KB page-aligned IVFFlat / HNSW**（NEW） | **DRAM (PG buffer) + PG storage (8KB pages, contiguous block alloc)** | **million-scale per instance** | **PG 单实例 + 应用分片** | **PG 全套 OLTP（ACID + WAL + replication）** |

### PASE 的 memory/disk 模式独特性（NEW）

[per systems/pase.md "8 KB page-aligned storage"]

PASE 在 wiki 已有架构中独有：
- **8 KB page-aligned**——PG 历史决策遗留；不像 SPANN 12-48 KB / DiskANN 4 KB SSD-aligned 是因 SSD I/O；PASE 8 KB 是 PG 内核 fixed
- **PG buffer + WAL + storage 三层完整 OLTP stack**——其他系统都是为 vector 重新设计；PASE 直接复用 PG 现成
- **Contiguous page block 分配**——专为 IVFFlat scan 优化；HNSW 不需要

→ PASE 的"内存 vs 磁盘"决策**完全由 PG 内核决定**——shared_buffers 设大则更多 page in DRAM；shared_buffers 小则更多依赖 OS page cache。是**PG-style 内存管理**，不是为 vector 定制。

### Cross-page storage 处理高维向量（NEW）

[per systems/pase.md "Cross-page storage"]

8 KB page payload ~7 KB；维度 > 2000（float32 = 8 KB）单 page 装不下：
- DataTuple cross-page：vector 切多 segment，每 page 存一段
- DataTuple header 记 segment offset

**对现代 768-d / 1024-d embedding** 影响：
- 768-d × 4B = 3 KB——单 page OK
- 1024-d × 4B = 4 KB——单 page OK
- 2048-d × 4B = 8 KB——边界（PageHeader / DataTuple header 后剩余可能不够）→ cross-page

→ PASE 默认支持现代 embedding，但 cross-page 影响 read-time multi-IO 开销。

### 与 ADBV 在 disk-vs-memory 维度的对比（NEW）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

ADBV 与 PASE 是 Alibaba ecosystem 同代但 host DB 不同：

| | ADBV | PASE |
|---|---|---|
| Host DB 内存模型 | OLAP MPP shared-nothing | OLTP RDBMS shared-buffer |
| Storage backend | Pangu 分布式 | PG 单实例 storage |
| Page size | 自定（不受 PG 约束） | **PG 8 KB fixed** |
| Quantization 路径 | ✓（VGPQ） | ✗（IVFFlat 全精度） |
| Distributed | ✓ | ✗ |
| 适用 scale | 13B production | million per instance |

### 已知盲区

- **PASE 应用层分片到多 PG 实例的工程实践**：论文未深入；推断需自实现 routing
- **Distributed PG (Citus / PolarDB) + PASE**：理论可行未实证
- **PASE 多实例联合 query（如 Federated PG）**：论文未涉及
- **2026 SIGMOD 跨 model 整合**：talk 当日 demo

## Cited Pages

- [systems/pase.md](../../systems/pase.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/pase-vs-cube-freddy.md](../../benchmarks/pase-vs-cube-freddy.md)
