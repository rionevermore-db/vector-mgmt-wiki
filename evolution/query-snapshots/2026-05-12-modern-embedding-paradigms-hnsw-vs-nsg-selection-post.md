---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: []
cited-count: 0
---

# Post-snapshot (modern-embedding-paradigms): hnsw-vs-nsg-selection

## TL;DR

无影响——graph-based ANN 算法选型与 embedding model 训练范式正交. 但需注意 embedding dim 影响 HNSW/NSG 构建效率: NV-Embed-v2 4096-dim vs GTE-base 768-dim, 4096-dim HNSW 构建 cost 5-6×, 间接影响 graph index 选型——大 dim 时 graph degree m 需 tune 否则 build memory 爆.

## Cited Pages

(无)
