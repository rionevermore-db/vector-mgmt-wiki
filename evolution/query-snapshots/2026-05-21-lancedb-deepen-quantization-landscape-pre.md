---
query-key: quantization-landscape
date: 2026-05-21
phase: pre
ingest-context: lancedb-deepen
wiki-pages-total: 106
cited-pages: [concepts/rabitq.md, concepts/product-quantization.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md, concepts/scann.md, concepts/vgpq.md]
cited-count: 5
---

# Pre-snapshot (lancedb-deepen): quantization-landscape

## TL;DR

当前 wiki quantization 覆盖**算法层很厚,production 采用层偏薄**:

- **PQ / OPQ / LSQ / ScaNN / VGPQ** [per concepts/product-quantization.md, scann.md, vgpq.md]:PQ 家族 + 各变体齐全。
- **RaBitQ** [per concepts/rabitq.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]:unbiased + sharp error bound + D-bit(PQ 一半)+ error-bound rerank 无需 K' 调参;6/6 dataset dominate PQ/OPQ/LSQ,6/6 优于 HNSW。
- **SQ(scalar quantization)**:wiki 内多处提及(Milvus IVF_SQ8 等)。

**production 采用是当前盲点**:RaBitQ 框定为学术 SOTA;[concepts/rabitq.md Open Questions] 把"Milvus 集成 RaBitQ"列为 *logical next step / 社区 PR* ——即 **wiki 当前没有明确的 RaBitQ production 采用 vendor 锚点**。"哪个 vendor 真在生产里跑 RaBitQ" 答不出具体名字。

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
- [concepts/scann.md](../../concepts/scann.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
