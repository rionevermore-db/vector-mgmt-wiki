---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [systems/diskann.md, systems/spann.md, systems/milvus.md, systems/spfresh.md, concepts/manu-ssd-hierarchical-kmeans.md, concepts/delta-consistency.md, topics/disk-vs-memory-ann.md, benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md]
cited-count: 8
---

# Post-snapshot (guo-2022-manu): memory-vs-disk-large-scale

## TL;DR (delta from pinecone-docs post)

**新增第 6 条 SSD 路线**：[Manu SSD-aware hierarchical k-means](../../concepts/manu-ssd-hierarchical-kmeans.md)——4KB block + DRAM cluster centers + **LSH-style 多次复制（4-8×）**。NeurIPS 2021 BigANN winner，比 baseline same QPS recall +60%。是 [SPANN closure clustering](../../systems/spann.md) 的 alternative 路线（同时期同问题，不同方案）。

## Answer

### 路线对比表完整版（updated with Manu SSD index）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 动态数据 | Quantizer |
|---|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ | 无 |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ | PQ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ | 无 |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ | 无 |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB peak | streamingMerge rebuild | PQ DRAM + 全精度 SSD |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB peak | 周期 rebuild | 无 |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ | SQ6 + HNSW coarse |
| DBMS LSM segment | DRAM + S3 | — | ~64 GB/节点 | LSM merge | segment 内固定 |
| DBMS cloud-native + zero-disk WAL | etcd + S3 | — | stateless | LSM + Streaming Node | segment 内固定 |
| 磁盘 + IVF + 全精度 + In-place 增量 | DRAM + raw NVMe | >0.86 @ ~5 ms | 持续 10 GB | LIRE in-place | 无 |
| SaaS + slab + adaptive indexing | Memtable + cache + slab on object storage | docs 未公开 | docs 未公开（auto） | slab merge | 用户不可见 |
| **DBMS + SSD-aware hierarchical k-means + LSH replication** | **DRAM (centers) + SSD (4KB block + 4-8× replication)** | **NeurIPS 2021 winner（baseline +60% recall）** | docs 未明示 | DBMS 全套（stream indexing + delta τ） | **SQ in 4KB blocks** |

### Manu SSD index vs SPANN closure clustering（NEW）

[per concepts/manu-ssd-hierarchical-kmeans.md "与 SPANN closure clustering 的对比"]

两者**同时期同问题不同方案**：

| | SPANN closure clustering | **Manu hierarchical k-means + LSH** |
|---|---|---|
| 复制粒度 | 选择性（仅边界向量） | **全数据集** |
| 复制系数 | 最多 8 replicas（边界） | **4-8× 全数据集** |
| 算法 | RNG-based pruning | LSH-style multi-tree |
| SSD 占用 | 节省（仅边界 8×） | 4-8× 总数据 |
| 实测 | Bing 千亿生产 | NeurIPS 2021 winner |
| 集成 | SPTAG | Milvus / Manu |

→ **同思想（边界向量需保护）不同实现**。SPANN 更省 SSD，Manu 更高 recall。

### Manu 把 update strategy 升级到 delta consistency 维度（NEW）

[per concepts/delta-consistency.md]

之前 wiki 的 update strategy 维度（in-place vs out-of-place）是**"何时 / 如何修改 index"**；Manu 加新维度：**"query 看到多旧的数据"**——delta consistency τ。

```
SSD 路线 + delta consistency 组合：
   Manu SSD index 写入 → memtable + WAL → 周期 flush → segment 4KB blocks
   Query τ=10s：写完 10s 内可见
   Query τ=0：等所有写入 sync
   Query τ=∞：用现有 SSD blocks，不等
```

→ SSD 路线下"数据可见性延迟"成可调参——之前所有 SSD 路线都是隐式 strong consistency 或 eventual。

### 与之前 ingest 的演进

| | xu-2023-spfresh post | pinecone-docs post | **guo-2022-manu post (NEW)** |
|---|---|---|---|
| SSD 路线数 | 4 | 5（+ Pinecone slab） | **6**（+ Manu SSD） |
| Update strategy 维度 | in-place vs out-of-place | + adaptive | **+ delta consistency τ** |
| Quantizer 在 SSD | PQ (DiskANN) / 全精度 (SPANN/SPFresh) | 不公开 | **+ SQ + LSH replication** |
| NeurIPS 实测 | n/a | n/a | **NeurIPS 2021 BigANN winner** |

### 已知盲区

- **Manu SSD index 在 Milvus v2.6.x 中的状态**：v2.6.x 文档列 DISKANN 而非 hierarchical k-means——可能已被 DiskANN 取代或仅在内核
- **Manu vs DiskANN 直接对比**：wiki 内未做（NeurIPS 2021 winner 数据是相对 baseline，不是 DiskANN）
- **Pinecone 内部是否用 Manu-style index**：闭源不公开
- **CXL / 持久内存**：仍未覆盖

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
- [concepts/delta-consistency.md](../../concepts/delta-consistency.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md](../../benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md)
