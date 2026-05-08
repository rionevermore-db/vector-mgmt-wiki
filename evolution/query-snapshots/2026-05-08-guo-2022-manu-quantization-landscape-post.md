---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [concepts/product-quantization.md, concepts/manu-ssd-hierarchical-kmeans.md, systems/milvus.md, topics/disk-vs-memory-ann.md]
cited-count: 4
---

# Post-snapshot (guo-2022-manu): quantization-landscape

## TL;DR (delta from pinecone-docs post)

**Manu (Milvus 2.x) 索引族表显式列出 PQ / OPQ / RQ / SQ + IVF-PQ / IVF-SQ / IVF-HNSW / IMI 全套**——比 Milvus 1.x 论文（仅 IVF-FLAT/SQ8/PQ）的 quantizer 范围大很多，比 Faiss 略少。**SSD-aware index** 用 **scalar quantization (SQ)** 存 4KB block；与 [DiskANN](../../systems/diskann.md) PQ 路线不同——Manu 信任 SQ（精度更接近全精度）+ LSH replication 而非 PQ。

## Answer

### Manu Table 1 索引族 quantizer 完整列表（NEW）

[per systems/milvus.md "Manu academic basis"]

| 类别 | Manu 列出 |
|---|---|
| Vector Quantization | **PQ / OPQ / RQ / SQ**（4 种） |
| Inverted Index | IVF-Flat / **IVF-PQ / IVF-SQ / IVF-HNSW** / IMI |
| Proximity Graph | HNSW / NSG / NGT（无 quantization） |

**对比 wiki 已有 source 的 quantizer 列表**：

| 系统 | PQ | OPQ | RQ | LSQ | SQ | RaBitQ | ScaNN |
|---|---|---|---|---|---|---|---|
| Faiss | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | (借鉴) |
| Milvus 1.x（SIGMOD） | ✓ (IVF_PQ) | ✗ | ✗ | ✗ | ✓ (IVF_SQ8) | ✗ | ✗ |
| **Manu (VLDB 2022)** | **✓** | **✓** | **✓** | ✗ | **✓** | ✗ | ✗ |
| Milvus v2.6.x | ✓ | ? | ? | ? | ✓ | ✗ | ✓ (SCANN) |
| DiskANN | ✓ (in DRAM) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| SPANN | ✗ (全精度) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| SPFresh | ✗ (全精度) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Pinecone | ✗ 用户不可见 | ✗ 用户不可见 | ✗ | ✗ | ✗ 用户不可见 | ? | ✗ |

→ Manu **明确扩展到 OPQ + RQ**——比 1.x 论文的 quantizer 范围更广。

### Manu SSD-aware index 用 SQ 而非 PQ（NEW）

[per concepts/manu-ssd-hierarchical-kmeans.md "Layer 1：4KB SSD-aligned blocks"]

Manu §4.4 SSD index 用 **scalar quantization (SQ)** 压缩存 SSD blocks：
- 不用 PQ——为什么？论文未明示，但推断：
  - SQ 单分量量化 → 精度接近全精度（相比 PQ 多倍失真）
  - SQ 解压速度 + 距离计算速度 ≈ 全精度（无 LUT 查询）
  - Manu 哲学：用 LSH-style 复制（SSD 容量增加）换 boundary recall——补偿 SQ 不如 PQ 紧凑的劣势

→ Manu SSD index = SQ + LSH replication 路线，与 DiskANN PQ + 全精度 re-rank 路线**对立**。

### 与之前 ingest 的演进

| | wang-2021 post | milvus-docs post | xu-2023-spfresh post | pinecone-docs post | **guo-2022-manu post (NEW)** |
|---|---|---|---|---|---|
| Milvus quantizer 列表 | IVF_FLAT/SQ8/PQ | + SCANN/DISKANN | 不变 | 不变 | **+ OPQ/RQ + IVF-HNSW/IMI** |
| SSD path quantizer | n/a | DiskANN PQ | SPFresh 全精度 | Pinecone 不公开 | **Manu SSD = SQ + LSH replication** |
| RaBitQ | 未覆盖 | 未覆盖 | 未覆盖 | 仍未覆盖 | 仍未覆盖 |

### Open / 未覆盖

- **Manu RaBitQ**：仍未覆盖
- **OPQ vs PQ vs RQ 在 Manu 实测**：Table 1 列出但 §5 未实测对比
- **SQ8H（Milvus 1.x SIGMOD）在 Manu 中是否保留**：Manu 论文 §4 提"refactored functionalities from Milvus" but specific to GPU SQ8H 未明示
- **Manu SSD SQ vs DiskANN PQ 对比**：两条路线哲学不同，但 wiki 内未直接 benchmark

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
- [systems/milvus.md](../../systems/milvus.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
