---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/acorn.md, systems/vespa.md, systems/weaviate.md, systems/qdrant.md]
cited-count: 6
---

# Post-snapshot (vespa-docs): hnsw-vs-nsg-selection

## TL;DR (delta from weaviate-docs post)

**Vespa 是 wiki 内第 4 个生产部署 HNSW 的 OSS vector DBMS**（Milvus + Qdrant + Weaviate + **Vespa**），NSG 在生产 vector DBMS 中**仍零覆盖**。**关键 NEW**：Vespa "Acorn-1" mode 是**[ACORN](../../concepts/acorn.md) 第 3 个独立 OSS production case**，与 Qdrant fallback + Weaviate correlation optimization 形成 **3-OSS ACORN production frontier 完全闭合**（Stanford 2024 学术 → 2026 三个独立 OSS DBMS 仅用 2 年）。**另一关键 NEW**：Vespa HNSW **嵌入 4-phase ranking pipeline**——与 BM25 / weakAnd / tensor compute 同列为 retrieval operator——是 wiki 内第一个把 HNSW 放进**复杂 ranking pipeline**而不是当作"主索引"的系统。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | Milvus + Qdrant + Weaviate (3) | **+ Vespa (4)** |
| NSG production | 仍仅 Taobao 2B | **不变** |
| ACORN production cases | 2 (Qdrant fallback + Weaviate correlation) | **3 (+ Vespa Acorn-1) — frontier 完全闭合** |
| HNSW + ranking pipeline 集成 | 仅"score + simple fusion" (Weaviate / Qdrant) | **Vespa 4-phase ranking (retrieval → first-phase → second-phase → global-phase ONNX)** |
| HNSW + filter mode 路径 | Qdrant Filterable HNSW; Weaviate ACORN+correlation | **+ Vespa 3-mode (pre-filter / post-filter / Acorn-1) YQL planner 选** |

### Vespa "Acorn-1" 与 ACORN frontier 闭合（NEW）

[per sources/docs/vespa/llms-full.txt §Approximate Nn Hnsw]

Vespa HNSW + filter 提供三种 mode：
1. **Pre-filter**：先 filter posting list 计算结果集，再 HNSW 仅在该集合内搜索（high selectivity 适用）
2. **Post-filter**：HNSW 全搜，结果后过滤（low selectivity 适用）
3. **Acorn-1**：Stanford 2024 ACORN 算法——HNSW graph 上**先 distance compute 再 filter check**，filter-aware traversal（mid selectivity 适用）

→ **3-OSS ACORN production frontier 完全闭合**：Qdrant v1.16.0 fallback + Weaviate correlation optimization + Vespa Acorn-1 mode。Stanford 2024 学术成果 → 2026 三个独立 OSS DBMS production deployment——**学术到主流仅 2 年**。

### Vespa HNSW 嵌入 4-phase ranking（NEW）

Vespa HNSW 不是独立"主索引"，而是 retrieval operator——与 BM25、weakAnd、tensor compute 同列：

```
retrieval (HNSW + filter + BM25) →
  first-phase (per-shard cheap score, all hits) →
  second-phase (per-shard expensive rerank, top 100) →
  global-phase (stateless container ONNX/cross-encoder, top 30)
```

→ **HNSW 是 wiki 内首次出现"嵌入复杂 ranking pipeline"**——Milvus / Qdrant / Weaviate 都是"HNSW score 返回 → application 层做 rerank"，Vespa **把 rerank pipeline 内置**。

### HNSW vs NSG 选择决策表（updated）

| 工程考量 | HNSW | NSG |
|---|---|---|
| Production OSS DBMS 覆盖 | **Milvus + Qdrant + Weaviate + Vespa + Faiss + VBASE + Pinecone** | NSSG + Taobao only |
| Filter-aware build / traversal | ✓ via ACORN (Qdrant + Weaviate + **Vespa Acorn-1**) / Filterable HNSW / Filter-aware planner | ✗ |
| Streaming insert+delete | HNSW α=1 fail → α=1.2 patch (FreshVamana Vamana base; **Weaviate HFresh** HNSW base preview) | ✗ |
| GPU-native | ✓ via CAGRA | ✗ |
| Ranking pipeline 集成 | **Vespa 4-phase ranking 首次工业化** | ✗ |
| Disk-resident segment | ✓ via Starling-HNSW | ✓ via Starling-NSG |
| Iterator + RM | ✓ via VBASE | ✗ |
| Web search heritage 整合 | **Vespa (Yahoo! 2003 lineage)** | ✗ |

### 选择决策（updated 2026-05-11 post vespa-docs）

- **CPU + 简单 single-vector TopK + million scale + 静态** → NSG
- **OSS vector DBMS production（主流）** → **HNSW** (Qdrant / Milvus / Weaviate / **Vespa** / Faiss)
- **复杂 ranking pipeline + tensor framework + ML rerank (ONNX/XGBoost)** → **Vespa** (HNSW 嵌入 4-phase ranking, 独占)
- **OSS Rust + 简洁 + filter heavy** → Qdrant
- **OSS Go + AI-native primary DB + agent stack** → Weaviate
- **OSS Go + 多 index_type + cloud-native** → Milvus
- **闭源 SaaS** → Pinecone
- **Streaming graph** → Vamana base (FreshDiskANN) OR HNSW base (Weaviate HFresh preview)

NSG 仍是"专攻 single-vector TopK 学术 benchmark"——production OSS vector DBMS **4 大 (Milvus + Qdrant + Weaviate + Vespa) 全部选 HNSW**。

### 已知盲区

- **NSG-based 主流 production DBMS**：仍未出现
- **Acorn-1 vs Qdrant Filterable HNSW vs Weaviate ACORN+correlation 性能对比**：3 OSS 实测 zero coverage
- **HNSW + 4-phase ranking 性能开销**：Vespa 引入 global-phase ONNX 的 P99 latency cost 未量化

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/acorn.md](../../concepts/acorn.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/qdrant.md](../../systems/qdrant.md)
