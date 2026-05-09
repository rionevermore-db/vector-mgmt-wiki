---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/vamana.md, concepts/block-shuffling.md, systems/starling.md]
cited-count: 5
---

# Post-snapshot (wang-2024-starling): hnsw-vs-nsg-selection

## TL;DR (delta from gao-2024-rabitq post)

**Starling 把 HNSW vs NSG 比较加上"disk-resident segment 友好度"新维度**——HNSW 多层结构与 in-memory navigation graph 抽象**天然 fit**（upper layers 直接当 nav graph，layer-0 走 disk + block shuffling）；NSG 单层 graph 需要单独构造 nav graph。Starling-HNSW / Starling-NSG / Starling-Vamana 三 variant 都 2× 各自 baseline framework——证明 graph 选择**不影响 Starling 优化路径的 portability**，但 HNSW 的实现优雅度最高。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| HNSW disk-resident 实证 | 不变（hnswlib in-memory only） | **+ Starling-HNSW 实证 disk-resident** |
| NSG disk-resident 实证 | 不变（NSG 主要 in-memory） | **+ Starling-NSG 实证 disk-resident** |
| Vamana disk-resident | DiskANN 默认 | **+ Starling-Vamana 加 block shuffling** |
| HNSW 多层结构 disk 适配 | n/a | **天然 fit Starling in-mem nav graph 抽象** |

### Starling-HNSW: HNSW 多层结构的天然 fit（NEW）

[per concepts/hnsw.md "Open Q" + systems/starling.md §6.7 + Fig 16]

HNSW 多层 graph 结构：
- Layer 0: 包含所有 vertex（base layer）
- Upper layers: 稀疏长程边（log-distributed）

Starling 的"in-memory navigation graph + disk-resident graph"抽象天然契合 HNSW：

```
Starling-HNSW:
   ┌──────────────────────────────────┐
   │  In-Memory                        │
   │  Layer 1, 2, ..., L (upper layers)│  ← 直接当 in-memory navigation graph
   │  + PQ codes                       │
   └─────────────┬────────────────────┘
                 ↓ from upper-layer entry
   ┌──────────────────────────────────┐
   │  On-Disk                          │
   │  Layer 0 (full vertex graph)     │  ← block shuffling 重排 layout
   └──────────────────────────────────┘
```

→ **HNSW 多层结构 = Starling 内存/磁盘划分的 free implementation**——不需要额外采样构造 nav graph（如 Starling-NSG / Starling-Vamana 需要采样 <10% vertex）。

实测 BIGANN 33M：Starling-HNSW QPS **>2× Disk-HNSW baseline**（[wang-2024-starling Fig 16(b)]）。

### Starling-NSG: 单层 graph 需采样（NEW）

[per concepts/nsg.md + Starling §6.7]

NSG 单层 graph，无 upper layers——必须**显式采样 <10% vertex** 构造独立 in-memory navigation graph。

Starling-NSG 实测：
- 同样 2× over Disk-NSG baseline
- 但实现复杂度比 Starling-HNSW 高（需独立 nav graph build step）

### Starling-Vamana: 默认配置（同 DiskANN）

[per concepts/vamana.md + Starling §3]

Starling 默认用 Vamana 作 disk graph——与 DiskANN 同算法选择。Block shuffling + in-memory nav graph 是增量。

### HNSW vs NSG 完整决策表（updated with Starling）

| 工程考量 | HNSW | NSG |
|---|---|---|
| 构建成本（in-memory）| 高 | **更低** |
| 单点查询性能（in-memory）| 中 | **更高** |
| Million-scale dataset 击败 | NSG 击败 | NSG 胜 |
| 增量更新 | ✓ | ✗ |
| Production 实证（in-memory）| Faiss / Milvus / hnswlib / Pinecone | Taobao 2B |
| Predicate-agnostic filter (HCPS) | ✓ via ACORN | ✗ |
| Iterator + RM 集成实证 | ✓ via VBASE | ✗ 未实证 |
| **Disk-resident segment 友好度** | **✓✓ 多层结构天然 fit Starling nav graph** | **✓ 需显式采样构造 nav graph** |
| **Starling-X 实证** | **Starling-HNSW 2× Disk-HNSW** | **Starling-NSG 2× Disk-NSG** |

→ Disk-resident 场景 HNSW 工程优雅度高；in-memory single-vector 场景 NSG 仍胜。两者使用场景**根本错位**。

### 选择决策（updated）

- **简单 single-vector TopK + million scale + 静态 + 内存富余 + in-memory** → NSG
- **single-vector TopK + 增量数据 + million scale + in-memory** → HNSW
- **single-vector TopK + 内存预算严** → IVF + RaBitQ（new alternative）
- **HCPS + memory + filter heavy** → HNSW + ACORN
- **复杂 multi-column / range / Join + SQL 用户** → HNSW + VBASE
- **billion-scale + memory** → HNSW + Faiss IVF coarse / RaBitQ + IVF
- **billion-scale + SSD + single-server** → DiskANN (Vamana) / SPANN
- **vector DBMS segment + disk-resident（NEW）** → **Starling-HNSW（推荐多层结构 fit）/ Starling-Vamana / Starling-NSG**

### 已知盲区

- **NSG / HNSW + iterator + RM 实证**：理论可行但 VBASE 仅集成 HNSW + IVFFlat + SPANN
- **Vamana + iterator + RM**：DiskANN + VBASE 集成是 logical next step
- **Starling + filter / multi-vector**：完全空白（Starling 仅 ANNS/RS）

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [concepts/block-shuffling.md](../../concepts/block-shuffling.md)
- [systems/starling.md](../../systems/starling.md)
