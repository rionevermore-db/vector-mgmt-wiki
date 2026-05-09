---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, concepts/product-quantization.md, concepts/rabitq.md]
cited-count: 3
---

# Post-snapshot (wang-2024-starling): quantization-landscape

## TL;DR (delta from gao-2024-rabitq post)

**Starling 不引入新 quantizer——仍用 PQ short codes**——但揭示一个 quantization 在 disk-resident graph 中的关键 use case："**routing-only quantization**"（用 PQ 估计邻居距离决定下一 hop，避免加载 full-precision vectors）。这是 quantizer landscape 中**与 distance-rerank 不同的应用模式**——不需 high accuracy，只需 directional correctness。**RaBitQ 替代 PQ 在此 use case 的潜在收益**：unbiased + sharp error bound 让 routing 决策更准 → 进一步减少 disk reads。两个 SIGMOD 2024 论文（Starling + RaBitQ）相互不知道，未实证联合效果——这是 wiki 内**新 frontier**。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| 新增 quantizer | RaBitQ | **无（Starling 用 PQ）** |
| Quantization use case | distance estimation + rerank | **+ routing-only quantization（NEW use case）** |
| Quantization in disk graph | DiskANN PQ DRAM + SSD rerank | **Starling PQ DRAM for routing, full-precision from disk for rerank** |
| RaBitQ + disk graph 实证 | n/a | **仍 zero coverage（Starling 与 RaBitQ 同年发表，相互不知）** |

### "Routing-only quantization" use case（NEW）

[per systems/starling.md §5.1 "PQ-based approximate distance"]

Starling 在 block search 中的 quantization 用法：

```
loop:
  load block B from disk (target vertex u)
  compute exact distance for u (full-precision)
  for each neighbor v in u's adjacency list:
      use PQ short code in DRAM to estimate dist(v, query)
      if v.PQ_dist > current threshold:
          skip v (don't load v's block)
      else:
          add v to candidate set
```

**关键观察**：
- PQ 只用于**routing 决策**（哪些 neighbor 值得 visit）
- **不用 PQ 计算最终 distance**（那是 full-precision 从 disk 读）
- → PQ accuracy 只需"directional correctness"（rough sort），不需 sharp error bound

→ 这与 [DiskANN](../../systems/diskann.md) 的"PQ for 距离估计 + SSD full-precision rerank for ranking"不同——DiskANN 把 PQ 当**主距离**用 + rerank 修正；Starling 把 PQ 当**纯 routing signal**用。

### Quantization use cases 完整划分（updated）

| Use Case | 准确度要求 | 适用 quantizer |
|---|---|---|
| 1. 主距离估计（in-memory） | 高（recall 受限）| Flat / SQ8 / SQfp16 |
| 2. 主距离估计（PQ-only） | 中（接受 60-70% recall） | PQ / OPQ / LSQ / VGPQ |
| 3. 距离估计 + rerank | 高（rerank 修正） | PQ / OPQ + SSD full-precision rerank |
| **4. Routing-only**（NEW） | 低（只需 directional） | **PQ short codes for graph traversal decision** |
| 5. Score-aware 距离估计 | MIPS 优化 | ScaNN anisotropic |
| 6. 严格 unbiased + error bound | 理论保证 | **RaBitQ** |

→ Starling exposes **use case 4** 作为 wiki 内首次显式区分——之前 use case 1-3 主导 quantization 讨论。

### RaBitQ 在 Starling routing-only use case 的潜在收益（NEW）

[per concepts/rabitq.md "Open Questions" + systems/starling.md "Open Questions"]

理论上 RaBitQ 替代 Starling 的 PQ for routing：

| 维度 | Starling-PQ (current) | Starling-RaBitQ (hypothetical) |
|---|---|---|
| Code length | 2D bits | **D bits（一半）** |
| Routing decision accuracy | biased estimate | **unbiased + sharp bound** |
| DRAM 占用（routing index） | reference | **1/2** |
| 错误 routing 概率 | 经验 | **理论 bound** |
| Disk reads 减少 | reference | **可能进一步减少**（更准 routing） |

**两个论文相互不知**：
- Starling [wang-2024]：SIGMOD 2024，用 PQ short codes for routing
- RaBitQ [gao-2024]：SIGMOD 2024，专注 in-memory ANN 实证，graph 集成 future work

→ **Starling + RaBitQ** 是 wiki 内 logical next step——但**实证未做**（甚至论文级 awareness 缺失）。

### Quantization landscape（不变）

[per concepts/product-quantization.md "后续演化"]

| 方法 | 类型 | 主要 use case | wiki coverage |
|---|---|---|---|
| Binary / SQ8 / PQ / OPQ / RQ / LSQ / ScaNN / VGPQ | PQ-family | distance estimation + rerank | full |
| ACORN compression | graph edges | neighbor list truncation | full |
| **RaBitQ** | hypercube + random rotation | distance estimation with theoretical bound | full |
| **Starling routing PQ** | use case 4 (NEW) | graph traversal routing decision | systems/starling.md |

### 已知盲区

- **Starling + RaBitQ 实证**：完全空白
- **Routing-only use case 在其他 disk graph 系统**：DiskANN 也用 PQ for routing 实质上类似，但论文未明示这个分类
- **Routing-only quantization 的最低准确度阈值**：不需 sharp bound 但需多准？未量化
- **Embedding model 升级 + Starling rebuild**：Starling per-segment build ~1200s（与 PQ build 同量级）；embedding 升级时 quantizer + graph + block shuffling 全部重做——无 incremental path

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
