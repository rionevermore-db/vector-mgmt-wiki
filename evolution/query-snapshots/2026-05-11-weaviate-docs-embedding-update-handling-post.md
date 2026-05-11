---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 2
---

# Post-snapshot (weaviate-docs): embedding-update-handling

## TL;DR (delta from qdrant-docs post)

**Weaviate ingest 不直接增加 embedding upgrade tooling**——llms.txt **未显式提及 Collection Aliases-style migration tool**（与 Qdrant 不同）。Weaviate 路径推荐 **named vectors**（同 collection 多 vector field，每 field 可独立 vectorizer 配置）——理论上可以做 model migration（创建新 named vector, vectorize 旧 data, 切换 query target_vector）但 docs 未明示 production workflow。**20 个 ingest 后 algorithm-level cross-model 仍 zero**。Weaviate 提供的工程便利仍属"工程 hack"——不像 Qdrant Collection Aliases 那样显式。

## Answer

### 与之前 ingest 的演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| Algorithm-level cross-model | 0 | **0**（20 个 ingest 后均确认） |
| Engineering tooling explicit | + Qdrant Collection Aliases atomic switch + Migration tool | **不变**（Weaviate llms.txt 未明示 model migration tool）|
| Named vectors model migration potential | (Qdrant 也有 named vectors) | **Weaviate named vectors first-class** |

### Weaviate Named Vectors 在 model migration 的应用（推断）

[per sources/docs/weaviate/llms.txt §Named vectors + concepts/qdrant.md "Collection Aliases"]

Weaviate **named vectors** 是 first-class feature：

```python
client.collections.create(
    "Article",
    vector_config=[
        Configure.Vectors.text2vec_weaviate(name="title", source_properties=["title"]),
        Configure.Vectors.text2vec_weaviate(name="body", source_properties=["body"]),
    ],
)
res = col.query.near_text("machine learning", target_vector="title", limit=3)
```

理论上 **named vectors 可以承载 dual-model deployment**：

```
Migration workflow (推断, docs 未明示):
  1. Existing collection "Article" with named vector "v1" (BERT embeddings)
  2. Add new named vector "v2" (SBERT embeddings, 不同 model)
  3. Background: re-embed existing docs to "v2"
  4. Query with target_vector="v2" 灰度切换
  5. Drop "v1" 当 verify OK
```

**对比 Qdrant Collection Aliases**：

| | Qdrant Collection Aliases | **Weaviate Named Vectors (推断)** |
|---|---|---|
| 抽象层 | Collection level (atomic alias swap) | **Vector field level** (per-collection 多 vector) |
| Atomic switch | ✓ (per Qdrant docs) | **未明示** (need 灰度 query target_vector) |
| Storage 翻倍 | Yes (两 collection 共存) | **Yes** (两 named vector 共存) |
| Schema flexibility | 新 collection 完全独立 schema | **同 collection** 内 named vector 可不同维度/distance |
| 显式 production migration tool | ✓ (qdrant-migration docker) | **✗** (docs 未提) |
| 文档明示 | "for upgrading neural network" | "for multi-representation (search by title vs body)" |

→ Weaviate named vectors **理论上可做 model migration**，但 docs 把这个 feature 定位为"multi-representation search"——**没有显式 production model migration tooling**。**Qdrant 在此维度仍更显式**。

### 仍是 zero algorithm coverage（不变）

[per 20 个 ingest 反复确认]

1. **跨 model embedding mapping function**：仍 zero
2. **Re-embedding 过程**：仍需全量 model inference
3. **Dimension 变化处理**：Weaviate named vectors 内可同 collection 多 dim/metric；Qdrant aliases 间可
4. **Model lineage tracking**：未涉及

### 跨 ingest 累积"embedding upgrade"状态

| Ingest | Algorithm 贡献 | Engineering 便利贡献 |
|---|---|---|
| 前 14 ingest | 0 | 0 |
| zhang-2023-vbase | 0 | 0 |
| gao-2024-rabitq | 0 | + 不需 KMeans 训练 |
| wang-2024-starling | 0 | + per-segment 滚动 rebuild |
| singh-2021-freshdiskann | 0 | + streaming insert during migration |
| ootomo-2023-cagra | 0 | + GPU rebuild 速度 |
| qdrant-docs | 0 | + Collection Aliases atomic + qdrant-migration tool |
| **weaviate-docs** | **0** | **+ Named Vectors first-class** (理论支持 dual-model 同 collection, 但 docs 未显式 production migration workflow) |

→ Algorithm 解仍 0；engineering 便利累积——Weaviate 在 named vectors 层提供潜在工具但**显式 production migration 弱于 Qdrant**。

### 已知盲区（仍未覆盖，20 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo——algorithm 解
- **跨 model mapping function**：仍 zero
- **Weaviate Named Vectors production model migration workflow**：docs 未明示
- **Weaviate Engram + model migration**：Engram (preview agent memory) 处理 conversation history 时如何 handle model 升级？docs 不深入
- **Weaviate Cloud + model migration tooling**：商业 SaaS 内部可能有但 docs 不公开

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
