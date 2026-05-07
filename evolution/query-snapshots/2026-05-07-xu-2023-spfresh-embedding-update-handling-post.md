---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [systems/spfresh.md, concepts/lire.md, topics/in-place-vs-out-of-place-updates.md, systems/milvus.md, systems/faiss.md]
cited-count: 5
---

# Post-snapshot (xu-2023-spfresh): embedding-update-handling

## TL;DR (delta from milvus-docs post)

**SPFresh 解决"vector-level update"但 NOT "embedding model upgrade"**——两个语义必须严格区分。[topics/in-place-vs-out-of-place-updates.md "与 embedding-update-handling 的区分"] 显式指出：SPFresh 处理**同 embedding 空间内**的 insert/delete/modify 单向量；**embedding model 升级**是 vector space 整体迁移，所有索引必须重建。SPFresh 的 in-place 增量更新**不能解决** embedding 升级问题，但是**双索引切换 cutover** 模式的强大基础设施。

## Answer

### vector-level update vs embedding model upgrade（NEW，本次 ingest 显式区分）

[per topics/in-place-vs-out-of-place-updates.md]：

| 维度 | vector-level update | embedding model upgrade |
|---|---|---|
| 范围 | 单个向量 insert / delete / modify | 整个 embedding space 切换 |
| 频率 | 持续（毫秒/秒级） | 偶发（季度/年级） |
| 索引语义 | 索引保持有效 | 索引完全失效 |
| 解决方案 | [SPFresh](../../systems/spfresh.md) LIRE / DiskANN streamingMerge / Milvus LSM | **wiki 仍未覆盖**（双索引切换 + re-embed pipeline） |

→ SPFresh 是**前者**的最优解；但 talk 当日 demo 主题（2026 SIGMOD 跨 model 整合论文）是**后者**——两件事。

### SPFresh 是 embedding 升级 cutover 的有用基础设施

虽然 SPFresh 不能直接解决 embedding model 升级，但其 in-place + 低资源占用使**新 collection cutover** 模式更可行：

> [推测，扩展 wang-2021 post 中的工业实践]

经典双索引切换需要：
1. 创建 new collection（用新 embedding model）
2. Backend batch re-embed + 写入 new collection
3. CDC 同步 incremental delta（[per systems/milvus.md] Milvus CDC 工具）
4. 流量切换

SPFresh 的贡献：**第 2 步 batch re-embed 写入可用 SPFresh in-place + 后续 CDC**——避免 new collection rebuild 高峰。但 collections 仍是双倍存储，cutover 仍需。

### Open Questions / 未覆盖

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo 主题，**仍未 ingest**
- **Embedding mapping function**：训练 small MLP 把旧 embedding → 新 embedding；仍 zero coverage
- **共享 embedding space 增量 fine-tune**：仍未覆盖
- **SPFresh 在跨模型场景的 reassign 失效**：SPFresh 假设 embedding space 稳定；如果 embedding model 升级，**centroid 与 vector 都失效**——SPFresh 不能 incremental migration

### 与之前两轮 ingest 的差异

| | wang-2021 post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| Vector update 解 | LSM | + CDC + function field | **+ in-place LIRE（最优）** |
| Embedding upgrade 解 | 双索引切换（手动） | + CDC 工具齐全 | **仍未解（基础设施更齐但 algorithmic 解仍 zero）** |
| 区分两个语义 | 隐含 | 隐含 | **显式区分** |

## Cited Pages

- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/lire.md](../../concepts/lire.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/faiss.md](../../systems/faiss.md)
