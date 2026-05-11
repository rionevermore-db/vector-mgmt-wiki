---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, queries/index-architecture-global-vs-routed.md]
cited-count: 2
---

# Post-snapshot (weaviate-docs): giga-scale-sharding

## TL;DR (delta from qdrant-docs post)

**Weaviate 与 Qdrant 都是 sweet spot at 中等规模 (1M-10B)**——千亿规模 docs 不公开实证。**对私有云配置的影响**：Weaviate 提供 **BYOC** (Bring Your Own Cloud) deployment——比 Qdrant Hybrid Cloud 更"客户全自管"。Weaviate **agent stack** (Query Agent + Engram) 是 Vector DBMS vendor 第一个在 OSS path 集成 RAG/agent 上层产品的 SKU——对千亿规模 RAG-heavy 应用是潜在价值，但 千亿 + agent stack 实证 wiki 内 zero。**4 个 OSS vector DBMS 全没千亿明示实证**：Milvus 300+ enterprise (具体 scale 不公开) / Qdrant Cloud (不公开) / Weaviate Cloud (不公开) / Pinecone (闭源 SaaS 不公开)。**千亿仍是 [DiskANN / SPANN / FreshDiskANN / SPFresh] 学术论文实证的 single-server / 大磁盘场景**。

## Answer

### 与之前 ingest 的演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| OSS DBMS 私有云 SKU | + Qdrant Hybrid Cloud (control plane @ Qdrant + data plane @ user) | **+ Weaviate BYOC + Dedicated** (similar pattern) |
| 千亿规模实证 | 不公开 | **不变** |
| RAG/agent stack 千亿 | 完全空白 | **+ Weaviate Query Agent + Engram** (vendor stack but production benchmarks 不公开) |
| Hybrid search (BM25 + vector) 千亿 | 完全空白 | **+ Weaviate hybrid search default + BlockMaxWAND** (千亿 benchmark 仍空白) |

### Weaviate 私有云部署选项（NEW）

[per sources/docs/weaviate/llms.txt + cost-performance-optimization.md]

| Weaviate SKU | 描述 |
|---|---|
| OSS Self-host | Docker / k8s / on-prem |
| Weaviate Cloud | 完全 managed |
| **BYOC (Bring Your Own Cloud)** | **客户 cloud account + Weaviate 管理** |
| **Dedicated** | Predictable throughput + isolation + compliance |

**与 Qdrant SKU 对比**：

| Vendor | OSS | Managed Cloud | Hybrid (control@vendor + data@user) | Customer-cloud | Edge |
|---|---|---|---|---|---|
| Qdrant | ✓ | ✓ | ✓ Hybrid Cloud | ✓ Private Cloud | ✓ |
| **Weaviate** | ✓ | ✓ | ✗ (没 explicit Hybrid) | **✓ BYOC** | ✗ |
| Milvus | ✓ | ✓ Zilliz Cloud | ✗ | ✗ | ✗ |
| Pinecone | ✗ | ✓ | ✗ | ✗ | ✗ |

→ **Qdrant 5 SKU vs Weaviate 4 SKU vs Milvus 2 SKU vs Pinecone 1 SKU**——Qdrant 在 deployment flexibility 上最广，Weaviate 略弱（无 Edge / 无 Hybrid Cloud-equivalent）但有 BYOC。

### 千亿 + 16 节点 × 1TB RAM 私有云：4 OSS DBMS 选择（updated）

[per queries/index-architecture-global-vs-routed.md + 推断]

| 决策因素 | Milvus | Qdrant | **Weaviate** | DiskANN/SPANN (library) |
|---|---|---|---|---|
| 千亿实证 | 300+ enterprise (具体不公开) | 不公开 | **不公开** | DiskANN single 1B / SPANN Bing 几千亿 |
| Index 多样性 | 多 (HNSW/IVF*/DISKANN/CAGRA) | HNSW only | **HNSW only + HFresh preview** | n/a (library) |
| Production engineering | 4-层 disaggregated | Rust 简洁 + Filterable HNSW | **AI-native primary DB + agent stack** | 算法 library |
| Hybrid search 内置 | SPARSE + reranker | sparse vectors | **`col.query.hybrid()` first-class** | n/a |
| Filter-aware | 5 strategies | Filterable HNSW + ACORN | **ACORN + positive/negative correlation** | n/a |
| 千亿 fit | 推荐 | 中等规模 sweet spot | **中等规模 sweet spot** | library 嵌入 |
| Vendor stack 千亿 | Zilliz Cloud | Cloud + Hybrid + Private + Edge | **+ Query Agent + Engram (RAG/agent)** | n/a |

→ **Weaviate 在 RAG/agent-heavy 千亿 workload 是潜在 unique value**——但 production benchmark 不公开。

### "AI-native primary DB" 千亿 use case 想象（NEW）

[per sources/docs/weaviate/llms.txt §Ideal Use Cases]

> Use Weaviate when: ... RAG / agentic retrieval with turnkey ingest + chunking + retrieval → Query Agent ... Long-lived agent memory → Engram

**千亿规模 RAG 应用** 推断：
- 千亿 docs index + Query Agent (turnkey PDF/auto-chunking/retrieval) + Engram (cross-session agent memory) + Hybrid search + ACORN filter — 完整 RAG stack
- 但 docs 不公开此规模实证；Pinecone Inference / Milvus + reranker 等也可拼装同等 stack

### 已知盲区

- **Weaviate 千亿规模 production**：完全空白
- **Weaviate BYOC 千亿 deployment**：完全空白
- **Weaviate vs Milvus / Qdrant / Pinecone head-to-head 千亿 benchmark**：四方都不公开
- **Weaviate Agent Stack + 千亿规模**：完全空白
- **HFresh preview + 千亿 streaming**：完全空白

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
