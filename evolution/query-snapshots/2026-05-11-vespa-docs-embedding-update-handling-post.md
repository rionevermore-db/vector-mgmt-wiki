---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [systems/vespa.md, systems/qdrant.md, systems/weaviate.md, systems/milvus.md, concepts/freshvamana.md]
cited-count: 5
---

# Post-snapshot (vespa-docs): embedding-update-handling

## TL;DR (delta from weaviate-docs post)

**Vespa 引入 wiki 内首个 "application package atomic deploy" model migration pattern**——schema + 索引 + ranking profile 作为**单一可部署单元** atomic upgrade，跨 embedding model 升级时 schema 演化（field 添加 / cell type 变更 / tensor 维度变更）走完整的 deploy-validate-rollback 流程，比 Qdrant Collection Aliases 抽象层级更高（package vs alias）。**关键 NEW**：Vespa 的**多 tensor field 共存**让"新旧 embedding 共存查询"成为 schema 默认能力（多 vector field 一等公民），无需 alias hack。**仍无算法层跨模型方案**——所有 ingest 后此 frontier 未关闭。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| 工程层 model migration tool | Qdrant Collection Aliases (atomic swap) | **+ Vespa application package atomic deploy** |
| 多 model 共存查询 | Weaviate "named vectors" per object | **+ Vespa multi-vector fields per doc (schema-level first-class)** |
| 算法层跨模型 compatibility | 仍 zero coverage（等 talk live demo SIGMOD 2026） | **不变** |
| Multi-vector indexing native | Weaviate named vectors | **+ Vespa `tensor<float>(i{},x[N])` map of vectors** |

### Vespa application package 视角（NEW）

[per sources/docs/vespa/llms-full.txt §Application Packages]

Vespa application package = `services.xml` + `schemas/*.sd` + ranking profiles + ONNX models + components——**单一可版本化、可部署、可回滚的单元**。Embedding model 升级走以下 pattern：

1. **新建 schema field**：`field embedding_v3 type tensor<float>(x[1536])`（保留旧 `embedding_v2`）
2. **新建 ranking profile**：`rank-profile v3-search` 引用新字段
3. **Deploy application package** — Vespa 自动 reindex 受影响字段、其他不变
4. **Query 切换**：YQL `using "v3-search"` 指定 ranking profile
5. **Validated 后旧 field 移除**：再次 deploy 新版本 drop `embedding_v2`

→ 比 Qdrant Collection Aliases 高一层抽象：Qdrant alias 切换的是**整个 collection**，Vespa application package 切换的是**整个集群行为**（schema + index + ranking + components）。

### Vespa multi-vector field 共存（NEW）

```
schema doc {
  field embedding_v2 type tensor<float>(x[768]) { indexing: index | attribute }
  field embedding_v3 type tensor<float>(x[1536]) { indexing: index | attribute }
  field embedding_multi type tensor<float>(i{},x[512]) {
    indexing: index | attribute
    index { hnsw { ... } }
  }
}
```

→ 一个 doc 同时持有：(1) 旧 model 768 维 + (2) 新 model 1536 维 + (3) chunk-level 多 512 维 vectors（matryoshka / RAG passage 风格）。**全部 first-class indexable + queryable**。其他 OSS vector DBMS 中：
- Weaviate "named vectors" 类似但 limit at named map 层
- Qdrant `multiple vectors per point` (v1.x+) 类似但是 dict 而非 tensor 矩阵
- Milvus 多 vector field 是 v2.x 加的，仍不如 Vespa tensor 形式 first-class

### 处理决策表（updated 2026-05-11 post vespa-docs）

| 场景 | 推荐方案 |
|---|---|
| **跨 embedding model 升级 + atomic switch + 集群行为整体变更** | **Vespa application package deploy + ranking profile 切换** |
| **跨 embedding model 升级 + collection-level 简单切换** | Qdrant Collection Aliases |
| **同一 doc 多 model embedding 共存 + complex ranking** | **Vespa multi-tensor field** |
| **同一 doc 多 embedding + 简单查询** | Weaviate named vectors |
| **大规模 stale segment 标记 + 后台 reindex** | Milvus / Vespa 都有 segment-level reindex pattern |
| **跨 model semantic preserve** | 仍未有 production——等 SIGMOD 2026 |

### 算法层跨模型方案——frontier 仍未关闭

[per sources/docs/vespa/]

Vespa docs 提到 "different embedding models" 仅在 application 层（应用如何选 model），**没有算法层 cross-model mapping**——与 Milvus / Qdrant / Weaviate 一致。Talk 当天 SIGMOD 2026 *Integrating Vector Databases across Embedding Models* live demo 仍是这道题的**唯一算法层答案**。

### 已知盲区

- **Application package deploy 时的实际 reindex cost**：Vespa docs 描述机制但未给 billion-scale doc 实测时间
- **Vespa vs Qdrant alias 切换 P99 stability**：head-to-head 未量化
- **Algorithm 层跨模型 mapping**：仍仅 SIGMOD 2026 live demo

## Cited Pages

- [systems/vespa.md](../../systems/vespa.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/freshvamana.md](../../concepts/freshvamana.md)
