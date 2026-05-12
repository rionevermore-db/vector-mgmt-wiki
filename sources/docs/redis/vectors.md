# Redis Stack Vector Search — Overview (WebFetch + 公知)

Source URL: https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/
Fetched: 2026-05-12
Acquisition: WebFetch 部分 + 业界公知补充 (RediSearch module 是 Redis Stack 核心组件)

## 架构

- **RediSearch** module = Redis Stack 核心 vector search engine
- In-memory searchable index 之上 Redis 数据结构
- 处理 vectorization, indexing, similarity search

## 索引类型

- **HNSW** (Hierarchical Navigable Small World) — ANN, recall/speed tradeoff 优化
- **FLAT** — 精确 brute-force, 高准确率, 大数据慢

## Hybrid Search

- **BM25 全文搜索** + vector similarity 组合 in single query
- Filter on 语义 relevance + keyword match
- Query syntax: `FT.SEARCH` 带 vector filters + text predicates

## Memory Model

- **全 in-memory**: 所有 index 在 RAM, sub-millisecond latency
- Disk 持久化 via RDB/AOF snapshot (不是 index-specific)
- **无 native disk-based index tiering** — wiki 唯一 in-memory-only vector vendor (vs DiskANN / SPANN / Pinecone 都 disk-aware)

## Scale 限制

- 单 index ~1B vectors (与 dimension 相关, RAM-bound)
- 单 Redis 节点 RAM 上限决定
- Horizontal scaling: Redis Cluster + 多 instance

## 部署 Pattern

```
Application → Redis Stack
  ├── Vector Index (HNSW/FLAT)
  ├── Cache Layer (string/hash)
  └── Full-text Index (BM25)
```

**典型 production**: 同一 Redis 实例同时做 application cache + vector retrieval —— 是 wiki 内唯一这种 dual-purpose vendor.

## Redis Stack vs Redis Cloud Vector Database

| Aspect | Redis Stack | Redis Cloud VectorDB |
|---|---|---|
| Deployment | Self-managed | Managed SaaS |
| Scaling | Manual (Cluster) | Auto |
| Features | Full RediSearch | Vector-optimized subset |
| Cost | Infrastructure | Pay-as-you-go |
