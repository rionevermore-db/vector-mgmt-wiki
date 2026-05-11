---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/turbopuffer.md, systems/qdrant.md, systems/weaviate.md, systems/vespa.md, systems/spfresh.md, concepts/lire.md]
cited-count: 6
---

# Post-snapshot (turbopuffer-docs): embedding-update-handling

## TL;DR (delta from vespa-docs post)

**Turbopuffer 引入 wiki 内首个"namespace-as-versioned-replica" embedding migration pattern**——`copy_from_namespace` API + atomic conditional writes 让"双 namespace blue/green 切换"成为 production-supported workflow，比 Qdrant Collection Aliases 多了 LIRE protocol incremental update（不需要 stop-the-world rebuild）。**关键 NEW**：Turbopuffer 100M+ namespace 设计点直接 enable "每 user/tenant 独立 embedding model" 极端 multi-model 部署——每 tenant 可有自己 model 版本，模型升级路径是 per-tenant rolling migration 而非 collection-wide。**算法层跨模型 zero coverage 仍未变**。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| 工程层 model migration tool | Qdrant Collection Aliases + Vespa application package | **+ Turbopuffer copy_from_namespace + atomic conditional writes** |
| 多 model 共存查询 | Weaviate named vectors + Vespa multi-vector field | **+ Turbopuffer per-namespace independent schema (100M+ namespaces)** |
| Per-tenant 独立 embedding model | wiki 未明示 | **Turbopuffer 设计点 enable 此极端模式** |
| 算法层跨模型 compatibility | 仍 zero coverage（等 talk live demo SIGMOD 2026） | **不变** |

### Turbopuffer namespace-as-versioned-replica 模式（NEW）

[per sources/docs/turbopuffer/llms-full.txt §write §pricing-log]

Turbopuffer 提供 `copy_from_namespace` API（2024-09 推出, **50% write cost discount**）：

```
# Migration pattern
1. ns_v2 = "embeddings-bert"      # old
2. ns_v3 = "embeddings-sbert"     # new, target
3. ns.write(copy_from_namespace=ns_v2, ...)    # copy raw docs (不带 vectors)
4. Re-embed via new model → upsert to ns_v3
5. Atomic alias / app config switch to ns_v3
6. ns.delete(ns_v2)
```

→ 比 Qdrant Collection Aliases 多了：
- **atomic conditional writes** for safe staging
- **per-document atomic batch** during ingest of new vectors
- **copy_from_namespace API** with 50% discount (encourages migration)

vs Vespa application package: 比 Vespa 更轻——Vespa 切换是 cluster-wide application package deploy；Turbopuffer 是 per-namespace independent，可逐 tenant rollout.

### Per-tenant 独立 model（极端 multi-model 部署）NEW

[per §multi-tenancy §limits + 设计 implication]

Turbopuffer namespace = S3 prefix (100M+ 一等公民) 直接 enable：
- 每 tenant 独立 namespace
- 每 namespace 独立 schema → 独立 vector dim / cell type
- → **每 tenant 可有自己 embedding model + version**

vs peer DBMS：
- Pinecone: index-level namespace（数百~数千 ceiling）→ 不支持 100M+ 独立 model
- Weaviate: collection-level multi-tenancy → 支持但 limit ~1M tenants
- Milvus: partition within collection → 不暴露 model-level 隔离
- Vespa: application package cluster-wide → 同 cluster 必须同 model
- **Turbopuffer**: per-tenant independent model 是**架构默认能力**

**Use case**：B2B SaaS 不同客户用不同 LLM provider (OpenAI / Anthropic / Cohere / open-source) 的 embedding，每客户独立 namespace → 独立 model → 独立 migration timeline.

### 处理决策表（updated 2026-05-11 post turbopuffer-docs）

| 场景 | 推荐方案 |
|---|---|
| **Per-tenant 独立 embedding + per-tenant rolling migration** | **Turbopuffer namespace-as-tenant + copy_from_namespace** |
| **跨 embedding model 升级 + atomic switch + 集群行为整体变更** | Vespa application package |
| **跨 embedding model 升级 + collection-level 简单切换** | Qdrant Collection Aliases |
| **同 doc 多 model embedding 共存 + complex ranking** | Vespa multi-tensor field |
| **同 doc 多 embedding + 简单查询** | Weaviate named vectors |
| **大规模 stale segment 标记 + 后台 reindex** | Milvus / Vespa 都有 |
| **跨 model semantic preserve algorithm** | 仍未有 production——等 SIGMOD 2026 |

### LIRE protocol 的 migration 含义（concept-level NEW）

[per concepts/lire.md + Turbopuffer SPFresh production]

Turbopuffer SPFresh 通过 [LIRE](../../concepts/lire.md) protocol incremental update——意味着即使在一个 namespace 内"逐渐替换 vector"（旧 model 删除 + 新 model 插入），cluster 持续 rebalance 不需要 stop-the-world rebuild。这与 Qdrant/Weaviate 的"双 collection alias swap"不同——是 **in-place** 渐进 migration.

**未完全验证**: 全 namespace 内 100% 旧 vector → 100% 新 vector 渐进替换的 recall 曲线 docs 未量化。

### 算法层跨模型方案——frontier 仍未关闭

Turbopuffer docs 不讨论算法层跨 model 兼容（与所有 peer DBMS 一致）。Talk 当天 SIGMOD 2026 *Integrating Vector Databases across Embedding Models* live demo 仍是唯一算法层答案。

### 已知盲区

- **copy_from_namespace 实测 throughput on giant namespace**：72 MB/s observed cap (per limits)
- **Per-tenant 100M+ namespace 实际 model-mix 案例**：Turbopuffer docs 不暴露客户具体配置
- **LIRE in-namespace gradual replace 实测**：docs 未给 recall 曲线
- **Algorithm 层跨 model mapping**：仍仅 SIGMOD 2026 live demo

## Cited Pages

- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/lire.md](../../concepts/lire.md)
