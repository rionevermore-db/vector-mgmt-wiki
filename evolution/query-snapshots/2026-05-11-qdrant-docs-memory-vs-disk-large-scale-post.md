---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (qdrant-docs): memory-vs-disk-large-scale

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 提供 wiki 内首个明确的 per-collection / per-vector / per-payload-field storage tier spectrum**——in-memory vs memmap (vector) + in-memory vs on-disk RocksDB (payload) + in-memory vs on-disk (HNSW index) + in-memory vs on-disk (payload index)——全部独立配置。这与之前 wiki 内 [DiskANN](../../systems/diskann.md) (固定 PQ DRAM + SSD 全精度) / [SPANN](../../systems/spann.md) (固定 centroids DRAM + posting SSD) 等"固定 hybrid 模式"形成对比。Qdrant 让用户**精细控制 memory/disk trade-off**——但需自己懂调参。

## Answer

### 与之前 ingest 的演进

| | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|
| Memory-disk hybrid 灵活度 | 固定 hybrid 模式 (DiskANN PQ+SSD / SPANN centroids+SSD / Starling segment) | **+ Qdrant per-component storage tier 独立配置** |
| 大规模 disk-resident | DiskANN / SPANN / Starling / FreshDiskANN / SPFresh | **Qdrant memmap (mmap) + on-disk HNSW** |
| GPU memory tier | CAGRA | **不变（Qdrant CPU only）** |

### Qdrant Storage Spectrum（NEW）

[per sources/docs/qdrant/manage-data/storage.md + manage-data/indexing.md]

| Component | Option 1 (default) | Option 2 (storage savings) |
|---|---|---|
| Vector storage | **in-memory RAM** | **memmap (mmap)**—mmap on disk |
| Payload storage | **in-memory** | **on-disk RocksDB** |
| HNSW index | in-memory | **on-disk** (`hnsw_config.on_disk = true`) |
| Payload index | in-memory | **on-disk** (v1.11.0+) |

**关键灵活性**：上述 4 个组件**独立配置**——可以"vector in-memory + payload on-disk + HNSW on-disk"等任意组合。

→ 这是 wiki 内**首次明确 per-component storage tier spectrum**——之前 DiskANN/SPANN 等都是"算法 + storage 模式绑定"的固定组合。Qdrant 解耦让用户精细 trade-off。

### Qdrant vs 固定 hybrid 系统的 trade-off（NEW）

| | DiskANN 固定模式 | **Qdrant per-component tier** |
|---|---|---|
| 设计哲学 | "算法 + storage 模式一起定" | "per-component 独立选择" |
| 用户配置复杂度 | 低（算法选项有限） | 高（4 个独立 storage 选项 × 多 collection） |
| Storage 利用率上限 | 算法预设 | **接近理论 optimum** (per-workload tuning) |
| 上手门槛 | 低（按算法 default） | 高（需懂 RAM/disk trade-off） |

→ Qdrant 是**"DIY storage tier"路径**——专家用户灵活但需懂；多数普通用户接受 default。

### Memory-disk landscape（updated with Qdrant per-component）

| 路线 | Hybrid mode | Memory:disk 比例 | 用户配置 |
|---|---|---|---|
| 全内存 (HNSW/NSG) | 100% RAM | 100:0 | 简单 |
| DiskANN | DRAM PQ + SSD 全精度 | 5:95 (approx) | DiskANN 算法预设 |
| SPANN | DRAM centroids + SSD posting | 16:84 | SPANN 算法预设 |
| Starling segment | DRAM nav + SSD reordered graph | per-segment | Milvus segment 模型预设 |
| FreshDiskANN streaming | LTI on SSD + TempIndex in DRAM | 12:88 (800M) | streaming 预设 |
| SPFresh + LIRE | DRAM cluster head + SSD | 1:99 (1B SIFT) | LIRE 预设 |
| CAGRA | GPU HBM only | n/a (GPU memory) | GPU memory only |
| **Qdrant (NEW)** | **per-component tier (4 options)** | **flexible (0:100 ~ 100:0)** | **用户独立配置** |

### Qdrant on-disk HNSW 的局限（NEW）

[per sources/docs/qdrant/manage-data/storage.md]

> 在 case of large payload values, it might be better to use OnDisk payload storage. ...In this scenario, **we recommend creating a payload index** for each field used in filtering conditions to avoid disk access.

→ Qdrant on-disk storage **依赖 page cache**——sufficient RAM 时几乎 in-memory 速度；不足时 disk I/O 显著 latency。不像 DiskANN 的"explicit beam-width control SSD reads"——Qdrant 把这个 trade-off 交给 OS page cache + 用户配置。

### 与 Qdrant Cloud 实际 production memory layout（NEW）

[per sources/docs/qdrant/cloud/]

Qdrant Cloud 提供 "Disk-based" 或 "Memory-based" 集群配置——用户在创建时选择：
- Memory-based: 极致性能，按 GB RAM 计费
- Disk-based: 成本优化，按 GB disk 计费 (memmap-heavy)

→ Qdrant Cloud 实际是"per-cluster memory/disk 比例"的 SKU 选择——比 OSS 更黑盒但用户友好。

### 已知盲区

- **Qdrant on-disk HNSW 在 large dataset 下的实际 latency curve**：docs 不公开 benchmark
- **Qdrant 4 个 storage component 联合最优配置**：4 个选项 × 多 collection scale 空间巨大
- **Qdrant vs DiskANN / SPANN 同硬件 head-to-head**：不公开
- **Qdrant + GPU CAGRA hybrid**：完全空白
- **Qdrant Edge 极小资源下的 storage tier 实证**：docs 仅 high-level

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
