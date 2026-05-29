---
query-key: quantization-landscape
date: 2026-05-29
phase: post
ingest-context: lance-blogs
wiki-pages-total: 107
cited-pages: [concepts/rabitq.md, concepts/product-quantization.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md, systems/lancedb.md]
cited-count: 4
---

# Post-snapshot (lance-blogs): quantization-landscape

## TL;DR

**NEW——RaBitQ-vs-PQ 首次有 production-vendor 实测数字(不再只有学术实现数)**:

LanceDB blog(2025-09)公布 IVF_RQ vs IVF_PQ vendor 自测 [per concepts/rabitq.md "Production 采用", sources/docs/lance-blogs-2026]:
- 1024-d 4KB → **~136 bytes**(1 bit/dim + 2 corrective scalar,~32× 压缩)
- **DBpedia 768d**:Recall@10 **96%+ vs IVF_PQ ~92%**,**495 QPS vs ~350**
- **GIST1M 960d**:**94% vs ~90%**,540-765 QPS vs ~420,**build 21s vs 130s(RaBitQ 还更快建)**

→ 与 RaBitQ 论文(6/6 dominate)方向一致:**recall + QPS + build time RaBitQ 全面优于 PQ**,现在有 production vendor 背书。**LanceDB 10B 分布式栈也用 RaBitQ**(配 HNSW-over-centroids 路由)——RaBitQ 不只单机量化,已进十亿级分布式 production。

quantization landscape 现状:PQ/OPQ/SQ(经典)+ RaBitQ(unbiased + error bound,**现有 2 vendor production + LanceDB 实测数字**)+ ScaNN(MIPS 原生);RaBitQ 是工业落地最快的新一代。仍 open:Milvus IVF_RABITQ 实测未公开;graph + RaBitQ(论文 future work)。

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)（Production 采用 + LanceDB 实测数字）
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
- [systems/lancedb.md](../../systems/lancedb.md)（§I + RaBitQ in 10B stack）
