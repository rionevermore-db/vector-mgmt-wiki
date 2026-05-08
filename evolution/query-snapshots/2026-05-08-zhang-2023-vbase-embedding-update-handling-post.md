---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [systems/vbase.md]
cited-count: 1
---

# Post-snapshot (zhang-2023-vbase): embedding-update-handling

## TL;DR (delta from patel-2024-acorn post)

**VBASE 不解决 embedding model 升级**——与所有 8 个已 ingest source 一致。**仍是 zero coverage**。VBASE 的 RM iterator 对 schema migration 提供**理论上轻微便利**：因为 RM 是 vector index 内在性质（不依赖具体 embedding model），重建索引时**不需要 retrain query engine layer**——但向量本身仍需重 embed。**全 wiki 50+ pages 后，cross-model embedding mapping 仍是绝对的 frontier 盲区**——这强化了 talk 当天 live demo 那篇 2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models* 的演示价值。

## Answer

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| Vector update 算法解 | + ACORN-1 incremental 友好（HNSW build cost） | **不变** |
| Embedding upgrade 算法解 | 仍 zero | **仍 zero**（8 个 ingest 后均确认） |
| Schema migration 工程便利 | per-system 各自处理 | **+ VBASE RM 抽象 query engine 层 stable** |

### VBASE 的"间接便利"（NEW）

[per systems/vbase.md + concepts/relaxed-monotonicity.md]

**Embedding model 升级**通常意味着：
1. 向量维度变（768 → 1024）→ 必须新 index
2. 距离分布变（余弦相似度阈值漂移）→ 应用层 threshold 重调
3. Filter schema 可能变（新增 filter 维度）

VBASE 在这三点上的影响：

| 维度 | 之前系统 | VBASE |
|---|---|---|
| 新 index | rebuild | rebuild（不变） |
| 距离阈值 | 应用层重写 | **可借 VBASE selectivity sampling 自动适应**（理论上） |
| 新 filter 维度 | per-system 重组 | **iterator 范式天然支持**（不需重建 query engine） |

→ VBASE 的 RM iterator 抽象**让 query engine 层 stable**——在 vector index 重建时，query engine layer / SQL syntax 不变。这是工程上的小便利，**但 embedding 本身仍需 reembed**（核心问题不解决）。

### 仍是 zero coverage 的核心问题（不变）

[per concepts/acorn.md "Open Questions"]

1. **Dimension 变化必须新 index**——仍 yes
2. **跨 model embedding mapping function**：zero coverage（VBASE 无新进展）
3. **Re-embedding 期间 storage 翻倍**：VBASE 没解决
4. **Filter set 演化**：[per acorn post] ACORN predicate-agnostic 适合动态 filter；VBASE iterator 也适合（多 vector index 与 filter 接口统一）

### 已知盲区（仍未覆盖，9 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **Production embedding upgrade**：所有 9 ingest 都 zero coverage
- **Dimension 不变但 model 变（768 → 768 from different model）下的索引兼容性**：理论上需 reembed，但**未量化"reuse same index"的 recall 退化**

## Cited Pages

- [systems/vbase.md](../../systems/vbase.md)
