---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, concepts/cagra-graph.md, topics/gpu-vs-cpu-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 4
---

# Post-snapshot (ootomo-2023-cagra): index-architecture-global-vs-routed

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 在 (a) global 单一索引路径上有特定 sweet spot——但仍受 GPU memory 限制不及千亿**：CAGRA single A100 ~100M (96-d) / ~24M (768-d) → (a) 路径在 100M-1B subset 下 GPU 上完全可行。**对 (c) 层次路由的影响**：(c) leaf 现在多一种 hardware option (GPU CAGRA per leaf segment for hot path)。**关键 insight**：之前 wiki 的 architecture 矩阵 (static/streaming × global/routed) 现在加 hardware 维度——**CPU vs GPU**——形成 8 种组合。CAGRA 在 GPU + global path (sweet-spot dataset) 下是 dominant；GPU + routed (hot subset of large cluster) 是新部署模式。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| (a) global 单一索引 实证 | + FreshDiskANN 800M-1B streaming | **+ CAGRA single-machine GPU memory budget** |
| (c) 层次路由 实证 | + per-segment streaming option | **+ GPU CAGRA hot-path leaf option** |
| Architecture × Update model 矩阵 | 4 种组合 | **8 种组合 (× CPU/GPU hardware)** |
| Hardware path | CPU + SSD | **+ GPU memory** |

### Architecture × Update Model × Hardware 矩阵（NEW）

[per queries/index-architecture-global-vs-routed.md + 累积 ingest]

之前 4 种组合升级为 **8 种组合**：

| | **CPU + SSD** | **GPU memory (NEW)** |
|---|---|---|
| **(a) Static + global** | DiskANN 1B / Faiss 1.5T mmap | **CAGRA single A100 ~100M (96d)** |
| **(c) Static + routed** | SPANN @ Bing / Milvus segment / Starling | **GPU CAGRA per hot leaf segment** |
| **(a) Streaming + global** | FreshDiskANN 1B / SPFresh 1B | **不可行**（CAGRA static, GPU memory 限制） |
| **(c) Streaming + routed** | per-segment FreshDiskANN / SPFresh | **不可行**（同上） |

→ GPU 路径仅适合 **static dataset**；streaming 仍 CPU-only。

### CAGRA 在 (a) global 路径的 sweet spot（NEW）

[per systems/cagra.md "Scale 边界"]

| Dataset size (96-d float32) | CPU + SSD options | **GPU CAGRA single A100** | GPU CAGRA Multi-GPU |
|---|---|---|---|
| ≤ 25M | overkill DiskANN | **best (low latency)** | overkill |
| 25M - 100M | DiskANN economic | **best** | overkill |
| 100M - 500M | DiskANN | needs FP16 / future PQ | **multi-GPU 4-8 cards** |
| 500M - 1B | DiskANN sweet spot | OOM | multi-GPU 8-16 cards |
| 1B - 千亿 | (c) routing required | OOM | **未实证** |
| 千亿+ | (c) routing 必须 | OOM | **不竞争** |

→ **CAGRA 在 25M-500M 为 (a) global 路径的 GPU dominator**——是 high-value 但 niche workload (e.g., 中等规模生产数据库, latency-critical retrieval)。

### CAGRA 在 (c) routed 的 leaf 角色（NEW）

[per concepts/cagra-graph.md + 推断]

(c) routed cluster 多 segment 配置下，每 leaf segment 是独立 index。CAGRA 可作 GPU-equipped leaf node:

```
Cluster routing layer
   ↓
Per-machine routing
   ↓
Per-segment index choice (NEW dimension):
   ┌────────────────────────────────────────────────┐
   │  Cold segment (low query traffic):             │
   │    CPU + SSD (DiskANN / SPFresh / Starling)   │
   ├────────────────────────────────────────────────┤
   │  Warm segment (medium traffic):                │
   │    CPU + SSD or GPU CAGRA (per-machine config) │
   ├────────────────────────────────────────────────┤
   │  Hot segment (high traffic, low latency):     │
   │    GPU CAGRA (NVIDIA A100 dedicated)           │
   └────────────────────────────────────────────────┘
```

→ **Hot/cold tiering with GPU+CPU hybrid** 是新部署模式——但跨 tier coordination 协议 wiki 内 zero coverage。

### 在 16 节点 × 1TB 私有云的具体决策（updated 2026-05-09）

[per queries/index-architecture-global-vs-routed.md + 推断]

**主流仍 (c) 层次路由 + per-machine 多 segment CPU + SSD**——CAGRA 不取代 cold/warm 路径。**新增**：

| 决策点 | 推荐 |
|---|---|
| Hot subset (≤5% data) | **+ GPU server (1-2 A100 each) running CAGRA** |
| Hot subset routing | query coordinator 维护 hot/cold lookup table |
| GPU memory budget for hot path | per-A100 ~24M (768-d float32) / ~48M (FP16) |
| Streaming insert into hot subset | CPU FreshDiskANN buffer → 周期 promote to GPU CAGRA |
| Migration: hot ↔ cold | 周级 rebuild GPU index from cold tier source |

→ GPU CAGRA 在千亿场景下是"hot path accelerator"而非"primary index"。

### 已知盲区

- **跨 hot/cold tier query routing 协议**：完全空白
- **GPU + CPU hybrid cluster 实证**：完全空白
- **(a) + GPU + 100M-1B 实证**：multi-GPU CAGRA 论文未深入
- **(c) + GPU + filter heavy**：完全空白
- **Streaming 在 GPU 上的可行性**：CAGRA 不支持，FreshCAGRA 完全 open
- **NVIDIA H100 / Blackwell 大规模实证**：A100 是当前实证上限

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
