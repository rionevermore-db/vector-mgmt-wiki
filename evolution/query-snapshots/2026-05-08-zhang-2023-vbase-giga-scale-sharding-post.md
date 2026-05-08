---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [systems/spann.md, systems/milvus.md, systems/vbase.md, topics/disk-vs-memory-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (zhang-2023-vbase): giga-scale-sharding

## TL;DR (delta from patel-2024-acorn post)

**VBASE 不直接影响千亿/万亿分片决策**——research prototype 只测到 Recipe1M 330K（million-scale）。但 VBASE+SPANN 集成 (§5.4) **首次实证 HNSW + SPANN 同 query engine 共存**——这对 giga-scale 多索引混部署有间接启示：iterator + RM 抽象让 in-memory graph 与 on-disk partition 同时活跃在同一 query engine，可能影响未来千亿混合架构的设计。**核心分片决策无变化**——SPANN partition + Bing 几千亿仍是工业实证；VBASE 是 query-engine layer 的进展，不是 storage-layer 的进展。

## Answer

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing 几千亿 + Faiss 1.5T | **不变** |
| Query engine 形态 | per-system 独有 | **+ VBASE 单 query engine 跨 HNSW + SPANN** |
| 多索引共存的可行性 | 工业上多系统 stack（Milvus + DiskANN）| **学术上单 PG instance + RM iterator 跨索引共生** |
| Filter heavy + giga | LCPS（FilteredVamana 28M DANN）/ HCPS 完全空白 | **不变** |

### VBASE+SPANN 在千亿场景的可推论价值（NEW）

[per systems/vbase.md "VBASE+SPANN" + zhang-2023-vbase Table 8]

VBASE Azure Standard_L16s_v3 NVMe 上集成 SPANN——单机 + million scale 实测。**理论上**这个 RM iterator 抽象可以扩展到分布式 SPANN（partition 跨 16 节点）+ 中心 query engine：

```
                ┌────────────────┐
                │ VBASE on PG    │
                │ + RM planner   │
                └───────┬────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ SPANN   │    │ SPANN   │    │ SPANN   │
   │ shard 1 │    │ shard 2 │    │ shard N │
   └─────────┘    └─────────┘    └─────────┘
```

→ **未实证**——VBASE 论文是单实例 PG。Cross-shard RM Phase 协调是开放问题。

### 16 节点 × 128U × 1TB RAM 的具体决策（不变）

[per queries/index-architecture-global-vs-routed.md 不变]：
- 千亿规模 → 路由式（SPANN-style partition + 单层 vector index per shard）
- 16 × 1TB RAM = 16 TB → 单层 inverted index 路径
- 向量 768-d float32 = 3 KB → 千亿向量 = 300 TB → 必须分片 + SSD 路线
- Week-level 全量更新 → out-of-place rebuild 可行（不需 LIRE 增量）
- P99 < 50 ms → SPANN 90% recall 时 ~1 ms 单机；分布式分片 32 partition → 每查询 ~6.3 节点 dispatch

VBASE 不改变这些决策。

### 已知盲区（仍未覆盖）

- **VBASE 千亿分布式实证**：研究原型，未涉及
- **HCPS + 千亿 + SSD**：完全空白（与 acorn ingest 一致）
- **Iterator 范式在 cross-shard 协调**：理论可行未实证
- **Bing 真实千亿配置 vs 私有云 16 节点**：私有云具体架构 wiki 仍 zero coverage

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/vbase.md](../../systems/vbase.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
