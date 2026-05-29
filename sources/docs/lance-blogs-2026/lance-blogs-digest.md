# Lance/LanceDB 技术 blog digest — Tier-1 (WebFetch, 2026-05-29)

> 来源:lancedb.com/blog 各篇。Layer-1 citation 锚点。

## GPU-accelerated indexing（`/blog/gpu-accelerated-indexing-in-lancedb`）

- **LanceDB 确有 GPU 加速索引构建**——具体是 **IVF 的 KMeans 聚类训练**(IVF_4096, L2, 1M 向量实测)。
- 硬件:**CUDA 或 Apple MPS**,`create_index(..., accelerator="cuda")` / `"mps"`;需装 pytorch(带 CUDA)。
- **production since LanceDB v0.3.3 / Lance v0.8.10**。
- 加速 **20-26×**:Linux L4 GPU 323s→12.5s;macOS M2 Max MPS 397s→21s。
- **PQ 训练尚未 GPU 加速**(roadmap:GPU PQ training + 向量 assign)。
- → 即:GPU 加速覆盖 **IVF KMeans 训练**,不含 PQ 训练、不含 graph build。这纠正"GPU index build 待核实"——它**真实存在**,只是 docs.lancedb.com indexing 页未提。

## 10B-scale 分布式（`/blog/how-lancedb-accelerates-vector-search-at-10-billion-scale`）

- **LanceDB Enterprise 分布式 compute-storage 分离**:10B 表切成 segment-level index,分布到多个 **Plan Executor**(各带 local SSD cache);query coordinator fan-out + merge top-k。
- 索引三层优化:(1) **IVF 分布式 build**(centroid 训练 + 向量 assign + encode 并行);(2) **HNSW over centroids**(query node 上快速找 nprobes,避免线性扫 centroid);(3) **RaBitQ 量化**(binary code + corrective + O(d log d) fast rotation)。
- **实测:10B 向量(1536-d,= 1B × 10 segment,10 节点):p50 18ms / p95 20ms / p99 21ms**,examined 20 partitions,top-100。
- Index build **比单节点快 5×**(10 个并行 indexing worker)。
- OSS 单节点瓶颈:单个大 index build 太慢 + 单节点 compute 不够 → Enterprise 加 segment 级粗粒度并行。

## RaBitQ 量化（`/blog/feature-rabitq-quantization`,2025-09-17）

- IVF_RQ:**1 bit/dim,~32× 压缩**(1024-d 4KB → ~136 bytes,含 2 个 corrective scalar + 随机正交矩阵 P)。
- **DBpedia 768d**:Recall@10 RaBitQ **96%+** vs IVF_PQ ~92%;throughput **495 QPS** vs ~350。
- **GIST1M 960d**:Recall@10 **94%** vs ~90%;**540-765 QPS** vs ~420;**build 21s vs 130s**(RaBitQ build 还更快)。
- 机器:Intel 12400F consumer PC。

## Lance format v2.2 benchmarks（`/blog/lance-format-v2-2-benchmarks-...`）

- 压缩(text-heavy):FineWeb 10M **比 Parquet 小 52%**(15,631 vs 32,534 MiB);OpenVid 1M 小 37%;FineWeb-1B 3.32→1.62 TiB(51%)。image-heavy(LAION)无改善(blob 已压缩)。
- blob random fetch **~75× faster than Parquet**(local NVMe);schema evolution 加列 **~61× faster**(Lance 13ms vs Parquet 520s);full scan S3 比 v2.0 快 23%。
- 机制:LZ4 on dictionary-encoded values;blob 存 dedicated position-indexed region(绕开 Parquet row-group 瓶颈)。

## Full-text search（`/blog/feature-full-text-search`,2025-08-11）

- **原生 FTS,弃用 Tantivy**("No more Tantivy!");具体算法(BM25/inverted)blog 未明示。
- **Hybrid search**:FTS + vector 结果合并 + rerank;统一 search 接口;`explain_plan` / `analyze_plan`。
- 实测(41M Wikipedia,8-GPU cluster):**ingestion 60,000+ docs/s,峰值 write 4 GB/s,41M 向量索引 30 分钟**。

## Late-interaction / multi-vector（`/blog/late-interaction-...`,2024-09-18）

- **LanceDB(截至 2024-09)不原生支持 ColBERT-style MaxSim late-interaction**——需外部自实现 MaxSim。
- 存储:multi-vector patch embedding 存成 flattened array + shape metadata(每文档 1030 patch × 128-d),**无专门 multi-vector index**。
- 模型:ColPali(SigLIP-So400m/14 + Gemma-2B)。query:全扫 30-34s,FTS/vector pre-filter top-100 后 ~6s。
- ⚠️ 日期较早(2024-09),可能已演进,需复核。
