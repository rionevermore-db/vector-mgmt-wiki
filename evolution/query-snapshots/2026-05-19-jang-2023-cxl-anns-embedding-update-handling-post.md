---
query-key: embedding-update-handling
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: []
cited-count: 0
---

# Post-snapshot (jang-2023-cxl-anns): embedding-update-handling

## TL;DR

**无影响**。CXL-ANNS 是静态数据集系统（NSG graph 预构建，pool manager 一次性 SSSP hop-count 布局），不涉及 embedding model 升级 / 索引重建 / 跨模型兼容。论文未触及 streaming insert/delete（已列入其 Open Questions——CXL 解耦内存上的增量更新类 FreshDiskANN/SPFresh 完全 open）。与 wiki 全 frontier 一致——embedding 升级 zero coverage。

## Cited Pages

(无)
