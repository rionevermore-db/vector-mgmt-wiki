---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/vamana.md, concepts/freshvamana.md, systems/freshdiskann.md]
cited-count: 5
---

# Post-snapshot (singh-2021-freshdiskann): hnsw-vs-nsg-selection

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN ingest 暴露 HNSW vs NSG 选择的一个根本盲点：α 参数**。HNSW 和 NSG 都隐式 α=1（aggressive RobustPrune）；FreshDiskANN [§3.3 Fig 1] 实测 SIFT1M 50 cycles × 5% delete+insert 下 HNSW recall **95% → 90%**、NSG recall **95% → 88%**——**两者在 streaming 场景下都失败**。这与 Vamana α=1.2（FreshVamana）下稳定 95%+ 形成对比。**对决策的影响**：之前 HNSW vs NSG 选择维度（增量 / 静态 / billion-scale / filter / iterator / disk）现在加上一个 **streaming 友好度**维度——**HNSW 和 NSG 都不友好**（都 fail），仅 Vamana 友好（via FreshVamana）。如果未来 hnswlib / nmslib 加入 α-augmented patch（理论可行），HNSW 才能进 streaming 场景。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| HNSW 参数已知维度 | M / ef / 多层结构 | **+ 隐式 α=1 在 streaming 场景失败** |
| NSG 参数已知维度 | R / L / C | **+ 隐式 α=1 同样问题** |
| Vamana 参数已知维度 | R / L / α (build perf) | **+ α (streaming 必要条件) — 升级地位** |
| HNSW vs NSG 对比 | + Starling-X disk-resident 友好度 | **+ streaming 友好度（都 fail）** |
| Streaming 实证基线 | n/a | **HNSW 95%→90%, NSG 95%→88% on SIFT1M 20 cycles** |

### α 参数的"再发现"（NEW）

[per concepts/freshvamana.md "α-RNG Property" + singh-2021-freshdiskann §4 Fig 3]

| 算法 | α 参数 | Static 性能 | **Streaming Recall (50 cycles 5% change)** |
|---|---|---|---|
| HNSW | implicit α=1 | best-in-class | **fail (95% → 90%)** |
| NSG | implicit α=1 | best-on-million-scale | **fail (95% → 88%)** |
| **Vamana α=1** | explicit α=1 | reference | **fail (95% → 89%)** |
| **Vamana α=1.2 (FreshVamana)** | explicit α=1.2 | 同 α=1 (slight cost) | **stable 95%+** |
| Vamana α=1.3 | explicit α=1.3 | slight cost | stable 95%+ |

→ **α 参数从"性能调节钮"升级为 streaming 必要条件**——这是 graph ANN 文献的根本 conceptual shift。

### HNSW / NSG α-augmented patch 的可行性（NEW）

[per concepts/freshvamana.md "Open Questions" + concepts/hnsw.md / concepts/nsg.md updated]

理论上：HNSW / NSG 加入 α > 1 RobustPrune 都可以解 streaming 失败问题——FreshVamana 仅是 Vamana 加 α=1.2。等价的"FreshHNSW"（hnswlib + α=1.2 RobustPrune）应该 work，但**工业实现没人做**。

| 算法 | α-augmented patch 复杂度 | 状态 |
|---|---|---|
| Vamana | trivial（α 是显式参数）| ✓ FreshVamana （Microsoft 实现） |
| HNSW | 中（需修改 RobustPrune，多层 graph 适配） | **未做**——community PR 候选 |
| NSG | 中（需修改 MRNG 边选择） | **未做** |

### HNSW vs NSG 完整决策表（updated）

| 工程考量 | HNSW | NSG |
|---|---|---|
| 构建成本（in-memory）| 高 | **更低** |
| 单点查询性能（in-memory）| 中 | **更高** |
| Million-scale 击败 | NSG 击败 | NSG 胜 |
| 增量 add-only | ✓ | ✗ |
| **Streaming insert+delete (50 cycles 5% change)** | **fail (95%→90%)** | **fail (95%→88%)** |
| Production 实证（in-memory）| Faiss / Milvus / hnswlib / Pinecone | Taobao 2B (static) |
| Predicate-agnostic filter (HCPS) | ✓ via ACORN | ✗ |
| Iterator + RM 集成实证 | ✓ via VBASE | ✗ 未实证 |
| Disk-resident segment 友好度 | ✓✓ 多层结构 fit Starling | ✓ 需独立 nav graph |
| Starling-X 实证 | Starling-HNSW 2× | Starling-NSG 2× |
| **α-augmented streaming-ready** | **✗ (理论可行未做)** | **✗ (同上)** |

### 选择决策（updated）

- **简单 single-vector TopK + million scale + 静态 + 内存富余 + in-memory** → NSG
- **single-vector TopK + add-only + million scale + in-memory** → HNSW
- **single-vector TopK + 内存预算严** → IVF + RaBitQ
- **HCPS + memory + filter heavy** → HNSW + ACORN
- **复杂 multi-column / range / Join + SQL 用户** → HNSW + VBASE
- **billion-scale + memory** → HNSW + Faiss IVF coarse / RaBitQ + IVF
- **billion-scale + SSD + single-server** → DiskANN (Vamana) / SPANN
- **vector DBMS segment + disk-resident** → Starling-HNSW / Starling-Vamana / Starling-NSG
- **billion-scale + streaming insert/delete + graph 路径（NEW）** → **FreshVamana / FreshDiskANN（HNSW / NSG 都不可用）**
- **billion-scale + streaming + cluster 路径（更经济）** → SPFresh + LIRE

### 已知盲区

- **HNSW α-augmented patch 实证**：理论 OK 未做
- **NSG α-augmented patch 实证**：同上
- **HNSW + RaBitQ + Streaming**：三层 frontier 完全空白
- **Vamana α=1.2 vs α=1.3 vs α=1.5 在不同数据分布下的最优**：经验调参 only

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [concepts/freshvamana.md](../../concepts/freshvamana.md)
- [systems/freshdiskann.md](../../systems/freshdiskann.md)
