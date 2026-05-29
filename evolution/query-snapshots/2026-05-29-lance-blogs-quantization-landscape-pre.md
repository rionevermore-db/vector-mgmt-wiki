---
query-key: quantization-landscape
date: 2026-05-29
phase: pre
ingest-context: lance-blogs
wiki-pages-total: 107
cited-pages: [concepts/rabitq.md, concepts/product-quantization.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md, systems/lancedb.md, systems/milvus.md]
cited-count: 5
---

# Pre-snapshot (lance-blogs): quantization-landscape

## TL;DR

RaBitQ production 采用已确认(LanceDB IVF_RQ + Milvus IVF_RABITQ),**但两家的 production 实测 recall/QPS 数字均未公开**——独立性能数仍只有 RaBitQ 论文(6/6 dominate PQ/OPQ/LSQ)。即:"RaBitQ 在 production 比 PQ 好多少"在 wiki 内**只有学术实现数,无 vendor 实测**。

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/milvus.md](../../systems/milvus.md)
