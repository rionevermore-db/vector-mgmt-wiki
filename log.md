# Log

> Append-only。每条记录：日期 | 操作类型 | 对象 | 影响概要。

```
2026-04-29 | init   | repo | 创建仓库骨架，写入 CLAUDE.md schema
2026-04-30 | ingest | sources/papers/malkov-2016-hnsw.pdf | 新建 concepts/hnsw.md, concepts/nsw.md, concepts/proximity-graph.md, benchmarks/hnsw-vs-faiss-200m-sift.md；更新 index.md, sources/README.md（含 PDF 重命名以符合命名约定）
2026-05-07 | ingest | sources/papers/jegou-2011-pq.pdf | 新建 concepts/product-quantization.md, benchmarks/pq-sift-recall.md；更新 hnsw.md / proximity-graph.md / hnsw-vs-faiss-200m-sift.md 的 related + 链接，更新 index.md, sources/README.md
2026-05-07 | ingest | sources/papers/fu-2017-nsg.pdf | 新建 concepts/nsg.md, benchmarks/nsg-vs-graph-anns-million.md；hnsw.md 同类对比表加 NSG 列 + Open Question；proximity-graph.md 变体表加 MSNET / MRNG 行；nsw.md / hnsw.md / proximity-graph.md 的 related 与 sources 都新增 fu-2017-nsg；更新 index.md, sources/README.md
2026-05-07 | ingest | sources/papers/guo-2019-scann.pdf | 新建 concepts/scann.md, topics/mips-vs-l2-nn.md（首个 topic！）, benchmarks/scann-glove1.2m-mips.md；product-quantization.md 加"后续演化：Score-aware loss"节，第一条 Open Question 标"部分回答"；hnsw.md / nsg.md 加 MIPS 相关 Open Question；三个 page 的 related/sources 都加 guo-2019-scann；更新 index.md, sources/README.md
2026-05-07 | ingest | sources/papers/johnson-2017-faiss-gpu.pdf | 新建 concepts/warpselect.md, topics/gpu-vs-cpu-ann.md（第二个 topic）, benchmarks/faiss-gpu-sift1b-deep1b.md；product-quantization.md 加"GPU 实现要点"节，第三条 Open Question 标"部分工程化"；proximity-graph.md 关键挑战补 GPU brute-force k-NN graph 反例；两个 page 的 related/sources 都加 johnson-2017-faiss-gpu；更新 index.md, sources/README.md。Faiss A 部分（GPU 算法），B 部分（库综述 Douze 2024）待后续 ingest
2026-05-07 | ingest | sources/papers/douze-2024-faiss-library.pdf | 新建 systems/faiss.md（首个 systems page！）, topics/index-selection.md（第三个 topic）, benchmarks/faiss-trillion-scale.md；product-quantization.md 加"Quantizer 家族中的位置"节（PQ/RQ/LSQ/PRQ/PLSQ 层级）；hnsw.md 典型实现补 HNSWlib 参考实现 + Faiss IndexHNSW；scann.md 加"工业影响"小节（Faiss FastScan 借鉴 ScaNN）；三个 page 的 related/sources 都加 douze-2024-faiss-library；更新 index.md（Systems 首次填充）, sources/README.md。Faiss A+B 序列完成
```
