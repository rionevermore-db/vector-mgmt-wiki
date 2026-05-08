---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/relaxed-monotonicity.md, systems/vbase.md]
cited-count: 4
---

# Post-snapshot (zhang-2023-vbase): hnsw-vs-nsg-selection

## TL;DR (delta from patel-2024-acorn post)

**HNSW 增加新维度："iterator-friendly via [Relaxed Monotonicity](../../concepts/relaxed-monotonicity.md)"**——VBASE [zhang-2023-vbase §4 + §6.2] 形式化 HNSW 的两阶段搜索（zoom-in 接近 → zoom-out 离开）为 RM 性质，让 HNSW 可以摆脱 TopK black-box 角色，作为 first-class iterator 在 query engine 中使用。NSG 同样满足 RM（论文 §4 提及 graph-based 方法都满足），但 VBASE 实测仅集成 HNSW + IVFFlat + SPANN——NSG 集成是开放。这给 HNSW vs NSG 选择**新增一个轴**：是否需要复杂 query（multi-column / range / Join）+ unified query engine。

## Answer

### 新增决策维度（NEW）

| 决策因素 | HNSW | NSG | VBASE 影响 |
|---|---|---|---|
| Iterator 接口可行性 | ✓ 实证（VBASE §6.2） | ✓ 理论上（满足 RM）但**未实证** | HNSW 在 unified query engine 中已 first-class |
| TopK 接口集成 | ✓（所有系统） | ✓ Taobao 部署 | HNSW 在新范式中是 wiki 内**唯一已实证**集成 |
| 复杂 query (multi-column / range / Join) | ✓ via VBASE | ✗ wiki 未覆盖 | VBASE 集成提供 HNSW 的"二代用法" |

→ HNSW 在新一代 query engine 范式（[Iterator + RM](../../topics/topk-vs-iterator-model.md)）下**已落地**；NSG 仍停留在 TopK 范式。

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| HNSW 改造方向 | + ACORN-γ predicate-agnostic | **+ VBASE iterator wrapping** |
| HNSW 工程接口 | TopK-only | **+ Open / Next / Close + amisrm** |
| HNSW 的 phase 性质 | 论文 §3 提及但未 expose | **§4 形式化为 RM；W=10 window 检测** |
| NSG 集成进展 | Taobao 工业部署不变 | **不变**（VBASE 未集成 NSG） |
| HNSW vs NSG 工程取舍 | 同前 | **+ "是否需 unified query engine"** 新轴 |

### HNSW vs NSG 完整决策表（updated）

| 工程考量 | HNSW | NSG |
|---|---|---|
| 构建成本 | 高（多层 + 双 entry） | **更低**（[fu-2017-nsg]） |
| 单点查询性能 | 中 | **更高** (NSG 论文) |
| Million-scale 击败 | NSG 击败 [per nsg-vs-graph-anns-million.md] | NSG 胜 |
| 增量更新 | ✓ HNSW add 支持 | ✗ NSG 静态 |
| 内存开销 | 多层 metadata | 单层更省 |
| Production 实证 | Faiss / Milvus / hnswlib / Pinecone (assumed) / **VBASE** | Taobao 2B [fu-2017-nsg] |
| **Predicate-agnostic filter (HCPS)** | **✓ via ACORN** [patel-2024] | ✗ wiki 未覆盖 NSG ACORN-equivalent |
| **Filter-aware build (LCPS)** | ✗（Vamana 路径独占） | ✗ |
| **Iterator + RM 集成实证** | **✓ via VBASE** [zhang-2023] | ✗ 未实证（理论可行） |
| **Unified query engine 中** | ✓ first-class iterator | ✗ |

→ HNSW 的"工程武器库"在 2024-2025 显著扩张：(predicate-agnostic via ACORN) + (iterator-friendly via VBASE)；NSG 在这两条进展线上都未跟上。

### 选择决策（updated）

- **简单 single-vector TopK + million scale + 静态数据** → NSG（最快 + 最省内存）
- **single-vector TopK + 增量数据 + million scale** → HNSW
- **HCPS + memory + filter heavy** → HNSW + ACORN
- **复杂 multi-column / range / Join + SQL 用户** → HNSW + VBASE（**NEW**）
- **billion-scale + memory** → HNSW + Faiss IVF coarse quantizer
- **billion-scale + SSD** → DiskANN (Vamana, 不是 HNSW 也不是 NSG) 或 SPANN（IVF）
- **filter-aware build + LCPS + SSD** → FilteredVamana (Vamana 改造)

### 已知盲区

- **NSG + iterator + RM 实证**：理论可行（NSG 满足 RM——单层 graph 同样有 zoom-in/zoom-out）但 VBASE 未集成。社区贡献空间
- **Vamana + iterator + RM**：[Vamana](../../concepts/vamana.md) 同样满足 RM；DiskANN + VBASE 集成是 logical next step

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/relaxed-monotonicity.md](../../concepts/relaxed-monotonicity.md)
- [systems/vbase.md](../../systems/vbase.md)
