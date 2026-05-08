---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [concepts/rabitq.md]
cited-count: 1
---

# Post-snapshot (gao-2024-rabitq): embedding-update-handling

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 不解决 embedding model 升级**——与所有 9 个已 ingest source 一致。**仍是 zero coverage**。RaBitQ 提供**轻微便利**：(a) RaBitQ index time 与 PQ 同量级（GIST 117s vs PQ 105s）→ rebuild 不比 PQ 慢，(b) RaBitQ 不需 KMeans 训练 → 重建逻辑更简洁，(c) RaBitQ 的 P 矩阵在新 model 下需重新采样（不能复用），但这与 PQ 的 KMeans codebook 不可复用是一致的。**全 wiki 现 55 pages 后，cross-model embedding mapping 仍是绝对的 frontier 盲区**——这继续强化 talk 当天 live demo 那篇 2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models* 的演示价值。

## Answer

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| Vector update 算法解 | + ACORN-1 incremental + VBASE iterator stable | **不变** |
| Embedding upgrade 算法解 | 仍 zero | **仍 zero**（9 个 ingest 后均确认） |
| Re-build 工程便利 | per-system 各自处理 | **+ RaBitQ 不需 KMeans 训练简化 rebuild 逻辑** |

### RaBitQ 在 model 升级时的"工程便利"（NEW）

[per concepts/rabitq.md "工程实现要点"]

**Embedding model 升级**通常意味着：
1. 向量维度变（768 → 1024）→ 必须新 index
2. 距离分布变 → 应用层 threshold 重调
3. Codebook / quantizer 训练须重做

RaBitQ 在第 3 项的差异：

| Quantizer | Rebuild 流程 | 时间 |
|---|---|---|
| PQ | 重新 KMeans 训练 codebook + 重新 quantize | GIST 105s |
| OPQ | 重新训练 rotation + KMeans + quantize | GIST 291s |
| LSQ | 重新 simulated annealing optimization | GIST **>24h** |
| **RaBitQ** | 重新 sample P 矩阵 + 重新 quantize（无 KMeans 训练） | GIST 117s |

→ RaBitQ 在 model 升级 + rebuild 时**比 OPQ 快 2.5×、比 LSQ 快 1000×、与 PQ 同量级**。这是工程便利但**embedding 本身仍需 reembed**——核心问题不解决。

### 仍是 zero coverage 的核心问题（不变）

[per concepts/acorn.md "Open Questions" + 9 个 ingest 反复确认]

1. **Dimension 变化必须新 index**——仍 yes
2. **跨 model embedding mapping function**：zero coverage（RaBitQ 无新进展）
3. **Re-embedding 期间 storage 翻倍**：RaBitQ 没解决
4. **Filter set 演化**：[per acorn post] ACORN predicate-agnostic 适合动态 filter；RaBitQ 与 filter 接口正交（quantizer 层不接触 filter）

### RaBitQ 的 P 矩阵 fixed 性质（NEW）

[per concepts/rabitq.md "工程实现要点"]

RaBitQ 用 random orthogonal matrix P 旋转 codebook。**关键约束**：P 一旦采样必须 fixed——不能在 model 升级时换 P，因为换 P 会改变所有 quantization codes（已存的 D-bit string 在 new P 下是无效的 quantization）。

但这不是限制——任何 quantizer（PQ KMeans codebook / OPQ rotation matrix）都有同样性质：codebook 一旦 train 必须 fixed。RaBitQ 不更糟。

**Open question**：Model 升级时是否能保留旧 P + 仅 normalize 新 embedding？理论上 unbiased 性质需要 ⟨P^{-1}o⟩ 的几何分布——新 model embedding 在旧 P 下分布可能偏移。论文未触及这个 cross-model 兼容性。

### 已知盲区（仍未覆盖，10 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **Production embedding upgrade**：所有 10 ingest 都 zero coverage
- **RaBitQ P 矩阵跨 model 复用**：理论开放（很可能不可行——分布会偏移）
- **Embedding 维度不变但 model 变（768 → 768 from different model）下 RaBitQ codes 是否仍 valid**：理论上 no（不同 model 的 768-d 分布不同）但未量化退化

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
