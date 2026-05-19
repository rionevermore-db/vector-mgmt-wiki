---
query-key: vector-scalar-bench-methodology
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: []
cited-count: 0
---

# Post-snapshot (jang-2023-cxl-anns): vector-scalar-bench-methodology

## TL;DR

**无影响**。CXL-ANNS 是 pure dense ANNS，无 attribute/scalar filtering，不涉及向量+标量混合查询的公平 benchmark 方法论。论文 eval 用 6 个纯向量 1B 数据集（BigANN/Yandex-T/Yandex-D/Meta-S/MS-T/MS-S），recall@k 0.9 target，无 selectivity 维度。filter-aware 与 wiki attribute-filtering 五策略框架联合是其 Open Q。

## Cited Pages

(无)
