# Log

> Append-only。每条记录：日期 | 操作类型 | 对象 | 影响概要。

```
2026-04-29 | init   | repo | 创建仓库骨架，写入 CLAUDE.md schema
2026-04-30 | ingest | sources/papers/malkov-2016-hnsw.pdf | 新建 concepts/hnsw.md, concepts/nsw.md, concepts/proximity-graph.md, benchmarks/hnsw-vs-faiss-200m-sift.md；更新 index.md, sources/README.md（含 PDF 重命名以符合命名约定）
2026-05-07 | ingest | sources/papers/jegou-2011-pq.pdf | 新建 concepts/product-quantization.md, benchmarks/pq-sift-recall.md；更新 hnsw.md / proximity-graph.md / hnsw-vs-faiss-200m-sift.md 的 related + 链接，更新 index.md, sources/README.md
2026-05-07 | ingest | sources/papers/fu-2017-nsg.pdf | 新建 concepts/nsg.md, benchmarks/nsg-vs-graph-anns-million.md；hnsw.md 同类对比表加 NSG 列 + Open Question；proximity-graph.md 变体表加 MSNET / MRNG 行；nsw.md / hnsw.md / proximity-graph.md 的 related 与 sources 都新增 fu-2017-nsg；更新 index.md, sources/README.md
```
