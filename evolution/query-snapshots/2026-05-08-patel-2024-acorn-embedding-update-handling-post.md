---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [concepts/acorn.md, concepts/filtered-vamana.md]
cited-count: 2
---

# Post-snapshot (patel-2024-acorn): embedding-update-handling

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**ACORN 不解决 embedding model 升级**——与所有已 ingest source 一致。**ACORN-1 incremental 友好** 性质让 cutover 比 ACORN-γ 工程上更可控（同 HNSW build cost 同时 search 时 2-hop 扩展近似 ACORN-γ）。**仍未解决核心**：embedding 维度变化必须新 index；跨 model mapping function 仍 zero coverage。**ACORN 论文 §8 未提 embedding update**。

## Answer

### ACORN-1 在 cutover 工程友好（NEW）

[per concepts/acorn.md "ACORN-1 算法"]

ACORN-1 与 HNSW 同构造（γ=1, M_β=M），只在 search 时做 2-hop neighbor expansion 模拟 ACORN-γ 的 dense graph。这意味着：

```
Embedding model 升级 cutover：
   1. 创建 new collection_v2 with ACORN-1
   2. 老 model 数据 → 新 model 重 embed → 增量 insert 到 collection_v2
   3. ACORN-1 build TTI ~HNSW（25.9 s for LAION 1M）→ cutover 窗口短
   4. 切流量
```

→ 比 ACORN-γ（10.5 小时 build for 25M）显著友好。ACORN-1 的"5× lower QPS"代价在 cutover 期可接受。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| Vector update 解 | + FilteredVamana incremental + Filtered-DiskANN A/B | **+ ACORN-1 incremental 友好（HNSW build cost）** |
| Embedding upgrade algorithmic 解 | 仍 zero | **仍 zero**（7 个 ingest 后均确认） |
| Cutover index TTI | FilteredVamana ~100s/M | **ACORN-1 ~HNSW（最快）** |

### 不解决的核心问题（与之前一致）

[per concepts/acorn.md "Open Questions"]

1. **Dimension 变化必须新 index**
2. **跨 model embedding mapping function**：zero coverage
3. **Re-embedding 期间 storage 翻倍**：ACORN-γ 1.3× HNSW 索引大小 + 双 collection → 双倍
4. **Filter set 演化**：升级 model 时如新增 filter 类型，FilteredVamana 需 partial rebuild；**ACORN predicate-agnostic 可保持 query 兼容**——这是 ACORN 优势在动态 filter 场景

### ACORN predicate-agnostic 在 dynamic filter 的潜在价值（推断）

> [推测，wiki 论文未直接覆盖]
>
> ACORN 不需要 construction 时知 filter set——意味着：
> - 模型升级 + 同时增加新 filter 类型 → ACORN index 不需重建（只 search 时改 query predicate）
> - FilteredVamana 必须 rebuild 来 incorporate 新 filter labels
>
> → ACORN 在 **filter schema 演化** 场景下比 FilteredVamana 更友好。但 embedding model 本身升级两者都需 rebuild。

### 已知盲区（仍未覆盖）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **ACORN production embedding upgrade**：学术 only，无工业实证
- **Dynamic filter schema 下的 ACORN vs FilteredVamana 实测**：理论 ACORN 优势未量化

## Cited Pages

- [concepts/acorn.md](../../concepts/acorn.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
