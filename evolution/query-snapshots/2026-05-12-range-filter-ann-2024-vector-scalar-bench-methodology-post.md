---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: [concepts/range-filter-ann-2024.md, topics/attribute-filtering.md]
cited-count: 2
---

# Post-snapshot (range-filter-ann-2024): vector-scalar-bench-methodology

## TL;DR

**重大 NEW** (vector + scalar bench methodology 显著扩展):

1. **Range filter benchmark 必须扫 range size**——3 paper 都强调 small/medium/large range 性能差异巨大. fair bench 应在 0.1% / 1% / 10% / 50% / 100% range 多档测试.

2. **Strategy A/B/C 经典 taxonomy 现 source-backed** (UNIFY §1):
   - Strategy A = Pre-filter (Alibaba ADBV / Milvus 分区)
   - Strategy B = Post-filter (Vearch / NGT)
   - Strategy C = Hybrid (SeRF / UNIFY)
   - Fair bench 应明确 vendor 用哪 strategy, 否则比较无意义

3. **2.29× SOTA 是 wiki 内 first OSS range filter ANN production-ready 数据点** (UNIFY)——之前 wiki 仅 ACORN categorical / Filtered-Vamana label 数据点.

4. **Production vendor 大概率全部 Strategy A**——Snowflake `@gte/@lte` / Databricks numeric filter / ES Lucene filter all 隐含 pre-filter pattern. SeRF / iRangeGraph / UNIFY 等 Strategy C **production vendor zero adopted**——是 academic-industry gap 重大数据点.

关键 NEW: filter-aware ANN bench methodology = (a) fix embedding model + (b) specify A/B/C strategy + (c) **range size sweep** + (d) 多 attribute combinations (PathFinder Ingest #13).

## Cited Pages

- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
