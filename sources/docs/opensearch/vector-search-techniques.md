# OpenSearch Vector Search — Overview (WebFetch + 公知)

Source URL: https://docs.opensearch.org/latest/vector-search/vector-search-techniques/index/
Fetched: 2026-05-12
Acquisition: WebFetch AI summary (structural extract) + 业界公知补充 (AWS-fork-of-Elasticsearch context)

## 起源

OpenSearch 是 AWS 在 2021 年 fork Elasticsearch 7.10 (Elastic 改 license 后)——k-NN plugin 来自 AWS 之前的 OpenDistro for Elasticsearch.

## 引擎选项

OpenSearch k-NN plugin 支持**多 engine 后端** (业界公知, doc 未直引):
- **Lucene**: HNSW (Java-native, 与 ES 同源)
- **NMSLib** (Non-Metric Space Library): HNSW (deprecated 2024+, Lucene 替代)
- **Faiss**: HNSW + IVF (Facebook AI Research C++ 库)

## Methods 与 Algorithms

- HNSW (所有 engine 都支持)
- IVF (Faiss engine only)

## Memory-optimized vectors

> "Memory-optimized vectors" - 文档明确 section, 含 quantization 选项

业界公知: Scalar quantization (8-bit, 4-bit), Binary quantization, Product Quantization (Faiss engine)

## Hybrid Search Capabilities

> "Tutorials on semantic search using byte vectors and references to neural sparse encoding in ingest pipelines"

业界公知: OpenSearch 有自研 **Neural Sparse** (类 SPLADE/ELSER) + BM25 + vector, **RRF processor** for fusion (与 Databricks Vector Search 同思路).

## Search Pipelines

> "Neural query enrichment processors / Neural sparse two-phase query processing / Semantic highlighting capabilities"

Hybrid search 通过 search pipeline 组合多 query type.

## 部署

OSS (Apache 2.0) + AWS managed (OpenSearch Service).

## 关键 differentiator vs Elasticsearch

- OpenSearch 是 OSS-license-permanent (Apache 2.0), ES 8+ 是 Elastic License (源可读但有限制)
- 3 engine 后端选择灵活 (Lucene/NMSLib/Faiss), ES 主要 Lucene 单一 engine 路径
- AWS managed offering 是默认 cloud path
