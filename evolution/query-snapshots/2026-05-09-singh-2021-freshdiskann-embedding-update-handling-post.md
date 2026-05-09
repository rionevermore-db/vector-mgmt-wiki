---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 2
---

# Post-snapshot (singh-2021-freshdiskann): embedding-update-handling

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 不解决 embedding model 升级**——与所有 12 个已 ingest source 一致。**仍是 zero coverage**。FreshDiskANN 的 streaming insert/delete 机制**仅适用于同 embedding 空间内的向量更新**，不适用于跨 model migration（model 升级时**全部 vector 要 re-embed → StreamingMerge 等价于 full rebuild**）。但 FreshDiskANN 的 **streaming-friendly architecture** 提供工程便利：可以 segment-by-segment 滚动 re-embed，旧 model segment 仍 serve queries，新 model segment 通过 FreshDiskANN streaming 接收 re-embedded 数据——这是 graceful migration 的天然 scaffold。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| Vector update 算法解 | + Starling segment-level rebuild graceful | **+ FreshDiskANN streaming insert during model migration** |
| Embedding upgrade 算法解 | 仍 zero | **仍 zero**（12 个 ingest 后均确认） |
| Model migration 工程便利 | + per-segment rolling rebuild | **+ FreshDiskANN streaming insert (no segment downtime)** |

### FreshDiskANN streaming insert 在 model migration 的应用（NEW）

[per systems/freshdiskann.md "Open Questions" + 推断]

Model 升级 segment-by-segment migration plan，**结合 FreshDiskANN streaming**：

```
For each segment in N:
  1. Old model segment (FreshDiskANN_old) 仍 serve queries
  2. 后台 re-embed segment 的 raw documents 用新 model
  3. New model segment (FreshDiskANN_new) 创建 + 用 FreshDiskANN streaming insert 接收
     re-embedded vectors（每秒 1800+ inserts，无 build downtime）
  4. 等 new segment 完整后 atomic swap
  5. 进入下一 segment
```

→ **FreshDiskANN streaming insert 比 Starling 周期 rebuild 更适合 migration**——streaming 期间 search 不阻塞，每秒持续接收 re-embedded vectors，整个 migration 期间 service 连续。

但 caveat：
- StreamingMerge 在 new segment build 期间需要 ~128 GB RAM
- 新旧 model segment **同时存在**期间 storage 翻倍
- Re-embedding pipeline（model inference）成本不被 FreshDiskANN 优化

### 仍是 zero coverage 的核心问题（不变）

[per 12 个 ingest 反复确认]

1. **跨 model embedding mapping function**：zero coverage
2. **Re-embedding 期间 storage 翻倍**：FreshDiskANN 没解决
3. **Dimension 变化必须新 index**：仍 yes
4. **Query model 升级 vs data model 升级不一致**：未涉及
5. **Old + new model 同时 serve queries 的语义**：完全开放

→ FreshDiskANN **解决 streaming 工程节奏**，不解决跨 model 语义。

### 跨 ingest 累积"embedding upgrade"状态

| Ingest | 算法贡献 | 工程便利贡献 |
|---|---|---|
| 前 9 个 ingest | 0 | 0 |
| zhang-2023-vbase | 0 | 0 |
| gao-2024-rabitq | 0 | + 不需 KMeans 训练简化 rebuild 逻辑 |
| wang-2024-starling | 0 | + per-segment 滚动 rebuild |
| **singh-2021-freshdiskann** | 0 | **+ streaming insert during migration（无 build downtime）** |

→ 13 个 source 后 algorithm-level 解仍为 0；engineering-level 便利累积。**真正 cross-model 解仍未出现**——继续强化 talk 当天 live demo 那篇 2026 SIGMOD 论文的演示价值。

### FreshDiskANN streaming + LIRE rolling migration 假想（NEW）

理论上**FreshDiskANN + SPFresh 联合 migration**：

```
Old SPFresh segment: 旧 model 数据，仍 serve queries
New FreshDiskANN segment: 新 model 数据，streaming insert
Atomic swap when new segment 完整

→ 利用 FreshDiskANN streaming insert speed (1800/sec)
→ 利用 SPFresh memory efficiency (4 GB) for read path
```

→ 这是 graph-path streaming 与 cluster-path in-place 的"protocol-level coordination"——wiki 完全空白。

### 已知盲区（仍未覆盖，13 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **FreshDiskANN multi-model concurrent serve**：理论上多 LTI（per model）共存可能；论文未涉及
- **Production embedding upgrade**：所有 13 ingest 都 zero coverage

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
