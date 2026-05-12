---
title: Redis Stack (RediSearch + Vector)
type: system
sources: [redis-docs]
related: [
  ./pinecone.md,
  ../concepts/hnsw.md,
  ../topics/disk-vs-memory-ann.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# Redis Stack (RediSearch + Vector)

**TL;DR**: Redis in-memory KV store 扩展的 vector search 能力, 通过 **RediSearch module** 实现 (Redis 8 起 default included BSD OSS). **核心 differentiator**: **全 in-memory, 无 disk-based index tiering**——是 wiki 内**唯一 pure-RAM-only 公开 vendor**, sub-millisecond latency 保证, 但 scale 受单节点 RAM 限制. **典型 production**: 同一 Redis 实例**同时做 application cache + vector retrieval** (dual-purpose), 是 latency-sensitive RAG 的独特 architectural 选择. 支持 HNSW (ANN) + FLAT (exact brute-force) + BM25 hybrid via `FT.SEARCH`.

## 架构图

```
        ┌─────────────────────────────────────────────────┐
        │  Application                                     │
        └────────────┬────────────────────┬───────────────┘
                     │                    │
                     │ Cache              │ Vector
                     │ (GET/SET)          │ (FT.SEARCH)
                     ▼                    ▼
        ┌─────────────────────────────────────────────────┐
        │  Redis Stack (single instance)                   │
        │  ┌────────────┐  ┌─────────────┐  ┌───────────┐ │
        │  │ String /   │  │ RediSearch  │  │ JSON /    │ │
        │  │ Hash       │  │ Module      │  │ Time-     │ │
        │  │ (Cache)    │  │  ┌────────┐ │  │ series    │ │
        │  │            │  │  │ HNSW   │ │  │           │ │
        │  │            │  │  │ Vector │ │  │           │ │
        │  │            │  │  └────────┘ │  │           │ │
        │  │            │  │  ┌────────┐ │  │           │ │
        │  │            │  │  │ BM25   │ │  │           │ │
        │  │            │  │  │ FT-idx │ │  │           │ │
        │  │            │  │  └────────┘ │  │           │ │
        │  └────────────┘  └─────────────┘  └───────────┘ │
        │                                                  │
        │  Persistence: RDB / AOF snapshot                │
        │  (data persistence, NOT index disk-tier)         │
        └─────────────────────────────────────────────────┘

        Scale: Redis Cluster for horizontal scaling
```

[per sources/docs/redis/vectors.md]

## 关键设计决策

| Decision | Trade-off |
|---|---|
| **全 in-memory, 无 disk index tier** | 优势: sub-ms latency 保证, **是 wiki 内唯一 explicit pure-RAM vendor**; 劣势: scale 上限严格被 RAM bound, 大规模 vector 成本高 |
| **HNSW + FLAT 两选项** | HNSW = ANN, FLAT = exact brute-force——FLAT 在小数据 (< 1M) 高准确 use case 仍 viable |
| **Dual-purpose deployment (cache + vector)** | 优势: latency-sensitive RAG 同 Redis 实例做 KV cache + retrieval, 减网络 hop; 劣势: vector index 占内存与 cache 争夺, capacity planning 复杂 |
| **`FT.SEARCH` 查询语法** | 优势: 与 RediSearch 全文搜索语法统一, vector + text + numeric filter 单查询; 劣势: 语法 unique to Redis, 不跨 vendor 迁移 |
| **Redis Cluster 水平扩展** | 优势: 单 instance RAM bound 可通过 cluster 突破; 劣势: cluster mode 跨 shard query 需 hash-tag routing, vector ANN 跨 shard 是 partition-routing 模式 (类 SPANN) |
| **Persistence via RDB/AOF (data 不是 index)** | 优势: 数据持久, restart 后从 disk 重建 RAM; 劣势: **vector index 仍需 cold-start 重建**, 重启 latency 高 (除非 RAM 直接 dump) |

## Scale 边界

[per sources/docs/redis/vectors.md]

| Axis | Limit |
|---|---|
| 单 index ~ vector count | ~1B (与 dim 相关, RAM-bound) |
| Latency | sub-ms (RAM-resident) |
| 单 Redis 节点 RAM | 物理上限 (typically ≤ 1 TB per server) |
| Horizontal scaling | Redis Cluster |
| Disk tier | **None** (NOT supported) |

> 实际部署: 768d float32 + HNSW (m=16, ~10× overhead) ≈ 32 KB/vec. 1 TB RAM ≈ 30M vectors raw + index. 用 int8 quantization 可 4×. 10亿 vector 需 ~330 GB int8, 单大 RAM node 可 fit.

## 与同类系统对比

[per sources/docs/redis/vectors.md 公知 + cross-system 比较]

| 与 Redis Stack 对比 | 共同 | 差异 |
|---|---|---|
| [Pinecone](./pinecone.md) | Managed SaaS, hybrid retrieval | Pinecone 专用 vector DB + disk tier; Redis 是 KV store + vector add-on, 全 RAM |
| [Faiss](./faiss.md) | In-memory ANN library | Faiss 是 library, Redis Stack 是 productized REST/protocol-based + persistence + cluster |
| [Vespa](./vespa.md) | Hybrid retrieval | Vespa hierarchical + disk + scalability 大得多; Redis pure RAM low-latency |

## 生产案例

> [推测, wiki 未覆盖]: Redis 部署量极大, 几乎所有 web app 用作 cache. 后期 vector 应用在 chatbot 短时记忆 / personalization / real-time recommendation 等 latency-sensitive RAG. Redis Cloud Vector Database 是 managed offering, AWS Marketplace 有列表.

## Redis Stack vs Redis Cloud Vector Database

[per sources/docs/redis/vectors.md]

| Aspect | Redis Stack | Redis Cloud VectorDB |
|---|---|---|
| Deployment | Self-managed | Managed SaaS |
| Scaling | Manual (Cluster) | Auto |
| Features | Full RediSearch (vector + text + JSON + time series) | Vector-optimized subset |
| Cost | Infrastructure | Pay-as-you-go |

## 关键 wiki 影响

1. **唯一 pure-RAM vendor**: wiki 内之前 disk vs memory 描述都默认 vendor 有 disk tier (DiskANN / SPANN / Pinecone / Vespa 都 disk-aware). Redis 是显式**no-disk-tier**, 是 latency-sensitive end of spectrum 的纯 in-memory 代表.

2. **Dual-purpose deployment pattern**: cache + vector in same Redis = **wiki 新 deployment pattern**——production RAG 把 retrieval 与 application cache 同 Redis 实例做, 减少网络 hop, 是 [LLM serving stack](../concepts/frontier-2025-distributed-vector-search.md) 中 vector search 的另一种 placement.

3. **HNSW + FLAT 双选项 = vendor-level brute-force baseline**: 大多 vendor 隐藏 brute force 选项, Redis FLAT 显式暴露——小数据精确召回场景 (e.g. 10k embeddings + 严格 recall) 是 brute force 经济选.

4. **RAM-bound scaling = production 上限**: Redis 上限作 wiki "pure-RAM ANN 现实 ceiling" 数据点——1 TB RAM × Redis Cluster 可达约 30-100M raw vectors 同时 quantize 后 1B 级别.

5. **典型 RAG 延迟最低端**: 与 Pinecone (~10-50ms) / Snowflake Cortex (~100ms) 对比, Redis sub-ms 是 latency-critical 路径——但 scale 反向受限.

## Open Questions

- **冷启动重建 index 时间**: 1B vector HNSW restart 后从 RDB 加载需多久? 是否 production-tested?
- **Redis Cluster vector ANN 跨 shard recall**: 跨 shard query 是 partition-routing (类 SPANN), 跨 shard recall fragmentation 多严重?
- **vs Redis Cloud Vector Database 性能**: Redis Stack self-hosted vs managed 同 workload 性能 / cost 真正差异?
- **HNSW + quantization compatibility**: Redis 是否支持 int8/binary HNSW vectors 与 ES/OpenSearch 同 level 量化? doc 未明示.
- **Redis 8 BSD license 变化对企业影响**: 之前 RSALv2 + SSPL, 现 BSD——商业 fork 现可行, 是否会催生 Redis fork 类 OpenSearch?
