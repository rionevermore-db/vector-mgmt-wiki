---
query-key: quantization-landscape
date: 2026-05-21
phase: post
ingest-context: lancedb-deepen
wiki-pages-total: 106
cited-pages: [concepts/rabitq.md, concepts/product-quantization.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md, systems/lancedb.md, systems/milvus.md]
cited-count: 5
---

# Post-snapshot (lancedb-deepen): quantization-landscape

## TL;DR

**NEW——RaBitQ 从"学术 SOTA"补上"production 采用"锚点,quantization landscape 的工业落地维度变实**:

1. **RaBitQ 已是 production index type,不只学术 quantizer** [per concepts/rabitq.md "Production 采用"]:
   - **LanceDB IVF_RQ** = RaBitQ "1 bit/dim"(1024-d 4KB → ~几百 bytes,`num_bits` 默认 1)[per systems/lancedb.md §C]
   - **Milvus IVF_RABITQ**(v2.6.x 索引族)[per systems/milvus.md]
   - 两家都走 **IVF + RaBitQ**(与论文主实证一致);填补了此前"vendor 采用列为 logical next step"的空白。

2. **PQ/OPQ/LSQ/ScaNN/SQ 算法层不变**,但现在能回答"工业上怎么用 RaBitQ"——答:作为 IVF 的量化层一等索引,与 IVF_PQ/IVF_SQ 并列可选(LanceDB 默认仍 IVF_PQ,dim≤256 时 IVF_PQ 常优于 IVF_RQ;极端压缩选 IVF_RQ)。

3. **量化轴与索引轴正交化更清晰** [per systems/lancedb.md §B]:partition 策略(flat IVF vs HNSW graph)× 压缩(PQ/RQ/SQ)= LanceDB 的 7 种 IVF* 组合——这是 quantization 与 graph/IVF 如何"配合使用"的具体 production 模板。

仍开放:两家 IVF_RQ/IVF_RABITQ **实测 recall/QPS 未公开**(production claim 成立但独立性能数仍只有 RaBitQ 论文 6/6 dominate);graph-based + RaBitQ 仍 future work。

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/milvus.md](../../systems/milvus.md)
