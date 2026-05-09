---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, concepts/block-shuffling.md]
cited-count: 2
---

# Post-snapshot (wang-2024-starling): embedding-update-handling

## TL;DR (delta from gao-2024-rabitq post)

**Starling 不解决 embedding model 升级**——与所有 10 个已 ingest source 一致。**仍是 zero coverage**。Starling §7 提"static disk index + 动态 in-memory + 周期 merge"模式（与 [Manu §3.5](../../concepts/manu-ssd-hierarchical-kmeans.md) 同源），适合 incremental data 但**不适合 model 升级（重 embedding）场景**。Starling 的 segment-level 设计反而**让 model 升级 in segment-by-segment 滚动更新成为可能**——比 single-server 大索引一次性重建更可控。但 wiki 内 11 个 ingest 后 cross-model embedding mapping function 仍是绝对 frontier 盲区。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| Vector update 算法解 | + RaBitQ unbiased estimator | **不变** |
| Embedding upgrade 算法解 | 仍 zero | **仍 zero**（11 个 ingest 后均确认） |
| Rebuild 工程便利 | + RaBitQ 不需 KMeans 训练 | **+ Starling per-segment 滚动重建模式** |

### Starling 的"per-segment 滚动 rebuild"潜在便利（NEW）

[per systems/starling.md "Open Questions" + §7]

Embedding model 升级常见挑战：
1. 整索引一次性 rebuild → 长时 service unavailable
2. Single-server 大索引 rebuild peak resource 远超 steady-state（DiskANN BIGANN 1B build 1100 GB peak vs 64 GB serving）
3. 多版本共存（旧 model + 新 model 双写期）storage 翻倍

Starling segment-level 形态的潜在解：

```
Segment-by-segment migration plan:
   for each segment in N:
     1. 旧 model segment 仍 serve queries
     2. 后台用新 model 重 embed segment 的所有 vector
     3. 用 Starling build pipeline 构造新 segment（per-segment 1200s）
     4. atomic swap 新 segment for old
     5. 进入下一 segment
```

→ **per-segment rebuild 是 graceful migration 的天然 unit**——比 single-server 大索引 rebuild 风险小、资源 peak 低。

### 但仍未解决的核心问题（不变）

[per 11 个 ingest 反复确认]

1. **跨 model embedding mapping function**：zero coverage
2. **Re-embedding 期间 storage 翻倍**：Starling 没解决
3. **Dimension 变化必须新 index**：仍 yes
4. **Query model 升级与 data model 升级不一致**（query 用 v2 / data 用 v3）：未涉及

→ Starling **只解决 rebuild 工程节奏**，不解决 mapping 问题。

### 跨 ingest 累积"embedding upgrade" 状态

| Ingest | 算法贡献 | 工程便利贡献 |
|---|---|---|
| 前 8 个 ingest | 0 | 0 |
| zhang-2023-vbase | 0 | 0 |
| gao-2024-rabitq | 0 | + 不需 KMeans 训练 |
| **wang-2024-starling** | 0 | **+ per-segment 滚动 rebuild** |

→ 11 个 source 后 algorithm-level 解为 0；engineering-level 便利零散增长。**真正 cross-model 解仍未出现**——这继续强化 talk 当天 live demo 那篇 2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models* 的演示价值。

### 已知盲区（仍未覆盖，11 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **Production embedding upgrade**：所有 11 ingest 都 zero coverage
- **Starling 多 segment 同时 in-flight upgrade**：理论可行未实证
- **Embedding 维度不变但 model 变下的 P 矩阵 / block shuffling layout 复用**：Starling 论文未涉及；理论上 layout 信息（vID → blockID）与具体 vector 数据正交，可能 reusable

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [concepts/block-shuffling.md](../../concepts/block-shuffling.md)
