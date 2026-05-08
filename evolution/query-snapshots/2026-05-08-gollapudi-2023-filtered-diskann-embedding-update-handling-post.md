---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [concepts/filtered-vamana.md, systems/diskann.md]
cited-count: 2
---

# Post-snapshot (gollapudi-2023-filtered-diskann): embedding-update-handling

## TL;DR (delta from yang-2020-pase post)

**Filtered-DiskANN 不解决 embedding model 升级**——与所有已 ingest source 一致。但 FilteredVamana 的 **incremental insert** 友好特性使 cutover 工程更简单：新 embedding model 数据可直接 insert 到新 collection 的 FilteredVamana index，不需要 batch rebuild（与 [SPANN](../../systems/spann.md) 周期 rebuild 路径相比更优）。**不解决核心**：embedding 维度变化必须新 index；跨 model mapping function 仍 zero coverage。

## Answer

### FilteredVamana incremental 对 cutover 的工程价值（NEW）

[per concepts/filtered-vamana.md "Alg 4: FilteredVamana Indexing"]

FilteredVamana 是 **incremental** algorithm（StitchedVamana 是 batch）：
- 新数据可直接走 GreedySearch + RobustPrune 加点
- 不需要全 rebuild

→ **embedding model 升级时**：
- 创建新 collection_v2 with FilteredVamana
- 老 model 数据导出 → 新 model 重 embed → 增量 insert 到 collection_v2 with FilteredVamana
- 期间 collection_v1 仍服务流量
- cutover

→ 与 SPANN（周期 rebuild）相比，FilteredVamana 让 v2 collection **从 zero 增量构建**——cutover 期 cost 更可控。

### 但仍不解决（与之前 ingest 一致）

[per concepts/filtered-vamana.md "Open Questions"]

1. **Dimension 变化必须新 index**
2. **跨 model embedding mapping function**：zero coverage
3. **完全 dynamic（删除）**：FilteredVamana **deletion 是 future work**——升级时可能需要先 delete old，但 FilteredVamana 不直接支持 delete
4. **Filter set 变化**：升级 model 时如果 filter schema 也变（新增 filter 类型），FilteredVamana 需要把新 filter 包含进 index——可能触发 partial rebuild

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| 索引 build time | PASE HNSW build 慢 | **FilteredVamana incremental + ~3× faster than StitchedVamana** |
| Cutover 工程 | PG schema migration | **+ FilteredVamana 增量 add 友好** |
| Embedding upgrade algorithmic 解 | 仍 zero | **仍 zero**（5 个 ingest 后均确认） |
| Deletion 支持 | PG 自身 | **FilteredVamana future work** |

### Filter-aware index 在 embedding 升级中的潜在新角色（推断）

> [推测，wiki 未覆盖]
>
> FilteredVamana 的 filter-as-baked-in-graph 机制可能用于 **embedding 多版本共存**：
> - filter A = "embedding model v1"
> - filter B = "embedding model v2"
> - 同 collection 含两 model 的 vector
> - query 时选 filter B 用 v2 model query
>
> → 缓慢迁移期可保留 v1 数据继续服务，新数据用 v2 + filter 区分。
>
> 但这是 hack——FilteredVamana 假设 filter 是 attribute meta（region、type 等），用做 embedding model 版本是非典型用法。论文未涉及此情景。

### 已知盲区（仍未覆盖）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **FilteredVamana deletion**：future work；与 [SPFresh LIRE](./lire.md) 在 graph 上的 in-place delete 可类比但未实现

## Cited Pages

- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [systems/diskann.md](../../systems/diskann.md)
