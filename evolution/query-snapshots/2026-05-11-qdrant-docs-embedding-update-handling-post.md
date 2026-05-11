---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 2
---

# Post-snapshot (qdrant-docs): embedding-update-handling

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant Collection Aliases 是 wiki 内首个明确的 production model migration tool**——通过 **atomic alias swap** 让新旧 embedding model collections 共存：旧 collection 仍 serve queries，新 collection 后台 build + ingest re-embedded data，原子切换 alias。**但仍不解决跨 model algorithm 问题**——新旧 model vector space 仍 incompatible，必须全量 re-embed。19 个 ingest 后 **algorithm-level coverage 仍是 0**，但**engineering tooling level 出现 first explicit support**。

## Answer

### 与之前 ingest 的演进

| | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|
| Algorithm-level cross-model | 0 | **0**（19 个 ingest 后均确认） |
| Engineering tooling explicit support | + per-segment rolling rebuild (Starling) / streaming insert (FreshDiskANN) / GPU rebuild speed (CAGRA) | **+ Qdrant Collection Aliases atomic switch (首个明确 production migration tool)** |
| Migration tool | n/a | **+ qdrant-migration docker image** |

### Qdrant Collection Aliases 的 model migration workflow（NEW）

[per sources/docs/qdrant/manage-data/collections.md "Collection aliases"]

```
Initial state:
  Collection prod_v1 (BERT embeddings)
  Alias `prod` → prod_v1
  User queries → alias `prod` → prod_v1

Migration steps:
  1. 在后台 build 新 collection prod_v2 (SBERT embeddings)
     - 完整 schema (维度可能变, distance metric 可能变)
     - 完整 indexes (HNSW + payload + ...)
  2. 用 qdrant-migration docker image 把 prod_v1 raw docs 经 SBERT re-embed → prod_v2 upsert
     docker run registry.cloud.qdrant.io/library/qdrant-migration qdrant \
       --source.url 'http://localhost:6334' --source.collection 'prod_v1' \
       --target.url 'http://localhost:6334' --target.collection 'prod_v2' \
       --migration.batch-size 64
  3. Atomic alias swap:
     POST /collections/aliases
     { "actions": [
         { "delete_alias": { "alias_name": "prod" } },
         { "create_alias": { "alias_name": "prod", "collection_name": "prod_v2" } }
     ] }
  4. Optional: delete prod_v1 (或保留作 rollback)

End state:
  User queries → alias `prod` → prod_v2 (atomic switch, no concurrent request affected)
```

→ **关键 engineering 价值**：
- **Atomic switch** (Qdrant docs 明示 "all changes of aliases happen atomically, no concurrent requests will be affected during the switch")
- **零 downtime** migration
- **Rollback friendly** (keep old collection until new is verified)
- **新旧 collection 可有不同 schema** (vector dim / distance metric 可变)

### 仍是 zero algorithm coverage（不变）

[per 19 个 ingest 反复确认]

1. **跨 model embedding mapping function**：仍 zero
2. **Re-embedding 过程**：仍需全量 model inference + storage 翻倍
3. **Dimension 变化必须新 index**：Qdrant aliases 工程上 handle this（新旧 collection 独立 schema），但 algorithm 上仍 yes
4. **Query model 升级 vs data model 升级不一致**：未涉及

→ Qdrant aliases **engineering level handle migration**，但 algorithm level **migration ≡ full re-embed + re-build**——本质问题不变。

### 跨 ingest 累积"embedding upgrade"状态（updated）

| Ingest | Algorithm 贡献 | Engineering 便利贡献 |
|---|---|---|
| 前 14 ingest | 0 | 0 |
| zhang-2023-vbase | 0 | 0 |
| gao-2024-rabitq | 0 | + 不需 KMeans 训练 |
| wang-2024-starling | 0 | + per-segment 滚动 rebuild |
| singh-2021-freshdiskann | 0 | + streaming insert during migration |
| ootomo-2023-cagra | 0 | + GPU rebuild 2.2-27× faster than CPU |
| **qdrant-docs** | **0** | **+ Collection Aliases atomic switch + Migration tool docker image (首个明确 production tool)** |

→ Algorithm 解仍为 0；engineering 便利累积。Qdrant 是**首个明确提供 cross-model migration production tool**——但本质问题（algorithm-level cross-model mapping）仍未解决。

### 已知盲区（仍未覆盖，19 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo——algorithm 解
- **跨 model mapping function**：仍 zero algorithm coverage
- **Qdrant aliases + 双 model concurrent serve**：理论可行（多 alias 指向不同 collection）但未实证 dual-model production
- **Qdrant migration tool 性能 production benchmark**：docs 不公开实际 throughput
- **维度变化 + 距离 metric 变化下的 Qdrant aliases**：理论上 prod_v1 用 cosine 768d, prod_v2 用 dot 1024d 可行（两 collection 独立 schema）；但 query side 必须知道版本切换

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
