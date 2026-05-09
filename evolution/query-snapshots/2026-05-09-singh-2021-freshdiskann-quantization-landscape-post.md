---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, concepts/product-quantization.md, concepts/rabitq.md]
cited-count: 3
---

# Post-snapshot (singh-2021-freshdiskann): quantization-landscape

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 不引入新 quantizer——但揭示 PQ 在 streaming 场景的稳定性问题**。FreshDiskANN StreamingMerge 用 PQ short codes 算 approximate distance（避免 LTI 全精度读）；初始 cycle 后 recall 略降（800M cycle 0 ~95% → cycle 20+ steady ~92.5%）但**不再下降**。**关键洞察**：PQ codebook 在 streaming 数据下漂移导致初始 2-3% recall loss——这是 wiki 内**首次明确量化** PQ 在 streaming 下的精度损失。**RaBitQ 替代 PQ 在 streaming 场景的潜在收益**：unbiased + sharp bound 让 codebook 不需 retrain（randomly transformed codebook 与 data 分布无关），理论上可保持 cycle 0 recall。两个 source 同期且互不知；联合实证 frontier。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| Quantization use case | distance estimation + rerank + routing | **+ streaming approximate distance during merge** |
| PQ 在 streaming 的稳定性 | 未量化 | **首次量化：~2.5% recall loss after 20+ cycles** |
| RaBitQ + streaming | 未涉及 | **理论可行（randomly transformed codebook 数据无关）** |
| 新 quantizer use case | n/a | **streaming routing**（不需要 high accuracy，只需 directional + stability） |

### Streaming 下 PQ codebook 漂移问题（NEW）

[per benchmarks/freshdiskann-streaming-sift800m.md "Result 8" + singh-2021-freshdiskann §5.5 Fig 4]

PQ codebook 在 build time 训练（k-means on data subset）；streaming 数据进来后**codebook 不重训**：

```
Cycle 0 (initial build): PQ codebook from initial 800M data
  Recall: 95%

Cycle 5 (after 30M inserts/deletes): same codebook, new data
  Recall drops slightly (codebook drift)

Cycle 20+ (steady state): same codebook, much shifted distribution
  Recall stabilizes at ~92.5% (3% loss)
```

→ FreshDiskANN 接受这个 trade-off（"3% loss for 5.25× faster merge"）；但揭示 **PQ codebook 的"实时数据分布无关性"假设是有限度的**——长期 streaming 后实际 distribution shift 引发 recall loss。

### RaBitQ 在 streaming routing use case 的潜在优势（NEW）

[per concepts/rabitq.md "Codebook" + 推断]

| 维度 | PQ codebook (Streaming) | **RaBitQ codebook (hypothetical Streaming)** |
|---|---|---|
| Codebook 训练数据依赖 | k-means on initial data → 数据漂移引发 recall loss | **数据无关（hypercube vertices + random rotation）** |
| Codebook 更新需要 | yes (周期 retrain) | **no（P 矩阵 fixed 即可）** |
| Streaming recall 稳定性 | 95% → 92.5% (cycle 20+) | **理论上 cycle 0 recall 维持** |
| 集成 streaming 系统 | DiskANN / FreshDiskANN | **未实证**（RaBitQ 论文不涉及 streaming） |

→ 理论上 **RaBitQ + FreshVamana** 是 logical 的 streaming + quantization 组合；wiki 内 zero coverage（两个论文 2021 / 2024 同期但相互不知）。

### Streaming routing use case 加入 quantization landscape（NEW）

[per systems/starling.md + systems/freshdiskann.md]

之前 wiki 内 quantization use cases（4 + 5 + 6）：
- 1. 主距离估计（in-memory）
- 2. 主距离估计（PQ-only with rerank）
- 3. 距离估计 + rerank
- 4. **Routing-only**（NEW from Starling — graph traversal decision）
- 5. Score-aware（ScaNN）
- 6. Theoretical-bound（RaBitQ）

**新增 use case 7**：**Streaming approximate distance for merge**

| Use Case | 准确度要求 | 适用 quantizer |
|---|---|---|
| 1. 主距离估计 in-memory | 高 | Flat / SQ8 / SQfp16 |
| 2. PQ-only 主距离 | 中 | PQ / OPQ / LSQ |
| 3. 距离 + rerank | 高 | PQ + SSD rerank |
| 4. Routing-only | 低 | PQ short codes for graph traversal |
| 5. Score-aware | 高 (MIPS) | ScaNN anisotropic |
| 6. Theoretical-bound | 严格 | RaBitQ |
| **7. Streaming merge approximate distance（NEW）** | **中 + 数据漂移耐性** | **PQ (current) / RaBitQ (hypothetical, 更稳定)** |

### Quantization landscape（updated）

| 方法 | 类型 | streaming 稳定性 | wiki coverage |
|---|---|---|---|
| Binary / SQ8 | scalar quantization | 数据无关，stable | indirect |
| PQ / OPQ / LSQ | product quantization | **codebook 漂移** → recall 缓慢 loss | full |
| ScaNN anisotropic | score-aware | 同 PQ（codebook 训练相关） | full |
| VGPQ | PQ + Voronoi 几何剪枝 | 同 PQ | full |
| ACORN compression | graph edges | n/a (graph layer) | full |
| **RaBitQ** | hypercube + random rotation | **理论数据无关 stable** | full |

→ **RaBitQ 在 streaming 场景的"codebook stability"潜在优势**——但需实证。

### 已知盲区

- **RaBitQ + FreshDiskANN 实证**：完全空白（两个论文相互不知）
- **PQ codebook 重训成本 vs streaming recall loss trade-off**：FreshDiskANN 接受 3% loss for 5.25× faster merge；其他系统的 trade-off curve 未量化
- **OPQ / VGPQ 在 streaming 下的稳定性**：仅 PQ 实证；OPQ 因加 rotation 可能更敏感数据漂移
- **Streaming + Score-aware (ScaNN) 在 MIPS 任务下**：完全空白

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
