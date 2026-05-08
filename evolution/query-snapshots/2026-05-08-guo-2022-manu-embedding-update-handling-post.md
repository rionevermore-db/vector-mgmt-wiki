---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [systems/milvus.md, concepts/delta-consistency.md, concepts/manu-ssd-hierarchical-kmeans.md]
cited-count: 3
---

# Post-snapshot (guo-2022-manu): embedding-update-handling

## TL;DR (delta from pinecone-docs post)

**Manu 论文 §7 future work 显式提"embedding generation toolbox" 但未实现**——明确未解决 embedding 升级问题。**新增的细粒度概念**：Manu §3.4 时间旅行（time travel）允许 user 指定物理时间 T 恢复——可作为"embedding 升级失败回滚"的工程支撑（rollback 到模型升级前的状态）。但**模型升级本身仍需新 collection + 双索引切换**。

## Answer

### Manu §7 提到的 embedding 集成 future work（NEW）

[per systems/milvus.md "Manu academic basis"]：

> "Embedding generation toolbox: For better application integration, we plan to incorporate an application-oriented toolbox for generating embedding vectors. This toolbox would incorporate model fine-tuning in addition to providing a number of pre-trained models that can be used out-of-the-box, allowing for rapid prototyping."

→ Manu 团队**明确意识**到 embedding lifecycle 是 wiki 的"first class concern"，但论文写作时未实现。**对应到 Milvus v2.6.x docs**：function-based field（用户在 schema 中指定 embedding 函数，Milvus 自动 embed）部分实现了这个 vision——但**仍未解决跨 model 升级**。

### Manu Time Travel 作为 rollback 支撑（NEW）

[per systems/milvus.md "Manu academic basis"]

```
Manu §4.3 Time Travel:
   - Periodic checkpoint（每 segment）
   - Per-segment progress map（哪些 WAL 已 apply）
   - User 指定 T → 加载 ≤T 的 closest checkpoint + replay WAL up to T
```

**对 embedding 升级场景的工程含义**：
- 切换到新 embedding model 后发现 recall 退化 → **time travel 回到升级前 collection 状态**
- 比 backup/restore 更细粒度（per-segment）
- 但**仍是 vector-level rollback**——不是 embedding-space migration

### 与之前 ingest 的演进

| | wang-2021 post | milvus-docs post | xu-2023-spfresh post | pinecone-docs post | **guo-2022-manu post (NEW)** |
|---|---|---|---|---|---|
| Vector update 机制 | LSM | + CDC + function field | + LIRE | + slab adaptive | **+ time travel rollback** |
| Embedding upgrade algorithmic 解 | 无 | 无 | 无 | 无 | **明确 future work（无）** |
| Time travel | 无 | 无 | 无 | 无 | **新增** |
| 学术承认 | 隐含 | 隐含 | 隐含 | 隐含 | **§7 显式 future work** |

→ **wiki 现有 source 中第一次有论文/团队显式承认 embedding lifecycle 是未解问题**。这强化了 **2026 SIGMOD Integrating Vector Databases across Embedding Models** 的重要性——它正是回答此 future work 的论文。

### Manu Delta Consistency 是否帮助 embedding 升级？

[per concepts/delta-consistency.md]

**No**——delta consistency τ 控制"vector update 多久可见"，不涉及"vector space 切换"。embedding 升级是 dimension / metric / 几何分布全变，超出 τ 适用范围。

但 **delta consistency + multi-collection** 的组合可支撑双索引 cutover：
- 旧 collection 服务（user 指定低 τ）
- 新 collection 后台 batch re-embed（high τ tolerance during build）
- 流量切换时同时切 collection name 与 τ

→ Manu/Milvus 提供基础设施齐全，algorithmic 答案仍 zero。

### 已知盲区

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo，仍未 ingest，Manu §7 future work 的对应学术答案
- **Manu 到 Milvus v2.6.x 之间 embedding 升级的工业实践**：CDC + function field 工具齐但 playbook 不成体系
- **跨 model embedding mapping function**：仍 zero coverage

## Cited Pages

- [systems/milvus.md](../../systems/milvus.md)
- [concepts/delta-consistency.md](../../concepts/delta-consistency.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
