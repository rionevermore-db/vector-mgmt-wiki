---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [concepts/hnsw.md, systems/chroma.md, systems/turbopuffer.md, systems/spann.md]
cited-count: 4
---

# Post-snapshot (chroma-docs): hnsw-vs-nsg-selection

## TL;DR (delta from pgvector-docs post)

**Chroma 引入 wiki 内首个 "ANN algorithm 与 deployment mode 绑定" 的 vendor**——Chroma OSS Core (single-node) 用 **HNSW**, Chroma Cloud (Distributed) 用 **SPANN**. 同 codebase 但**架构不同**, 选 ANN 由 deployment mode 自动决定. **关键 NEW**: HNSW production OSS deployment 从 5 增到 6 (Milvus + Qdrant + Weaviate + Vespa + pgvector + **Chroma OSS Core**); SPANN production deployment 从 3 增到 4 (Microsoft Bing 历史 + Vespa OSS + Turbopuffer SPFresh 衍生 + **Chroma Cloud**).

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 5 | **6 (+ Chroma OSS Core)** |
| SPANN production deployment | 3 (Bing历史 + Vespa OSS + Turbopuffer SPFresh) | **4 (+ Chroma Cloud)** |
| ANN choice 与 deployment mode 绑定 | 未涵盖 | **NEW: Chroma OSS=HNSW / Cloud=SPANN, 同 codebase** |

### Chroma OSS=HNSW + Cloud=SPANN 双绑定哲学（NEW）

[per sources/docs/chroma/llms-full.txt §Cloud + §Index Configuration]

**OSS Chroma Core (single-node)**:
- Default ANN: HNSW (graph)
- Reason: single-node fit, no distributed coordination needed

**Chroma Cloud (Distributed)**:
- Default ANN: **SPANN** (centroid + posting on object storage)
- Reason: object-storage-based persistence, SPANN minimizes object storage roundtrips
- Same algorithm choice 作 Turbopuffer (SPFresh) 同 philosophy: 选 centroid-based ANN for object storage primary

→ **架构-ANN 绑定**: Chroma 通过 deployment mode 自动决定 ANN, dev (HNSW) → prod (SPANN) migration 不暴露给用户.

### HNSW vs NSG vs SPANN 选择决策表（updated 2026-05-12 post chroma-docs）

| 工程考量 | HNSW (6 OSS vendor) | NSG | SPANN (4 production) |
|---|---|---|---|
| Production OSS DBMS deployment 数 | **6 (Milvus + Qdrant + Weaviate + Vespa + pgvector + Chroma OSS)** | 2 (NSSG + Taobao only) | 4 (Bing历史 + Vespa OSS + Turbopuffer SPFresh + **Chroma Cloud**) |
| Single-node dev friendly | ✓ | partial | ✗ (需 distributed infra) |
| Object storage primary friendly | ✗ | ✗ | **✓ (Turbopuffer + Chroma Cloud both)** |
| Filter-aware (ACORN-style) | ✓ (4 OSS implementations) | ✗ | partial via Vespa Acorn-1 |
| GPU-native | ✓ via CAGRA | ✗ | ✗ |
| Streaming + α=1.2 patch | HFresh / FreshVamana | ✗ | SPFresh (LIRE) |

### 选择决策（updated 2026-05-12 post chroma-docs）

- **Dev / Prototype + AI-agent-native + OSS** → **Chroma OSS Core HNSW**
- **Cloud serverless + object storage primary + AI-agent (MCP)** → **Chroma Cloud SPANN**
- 已部署 Postgres + 加 vector → pgvector HNSW
- 大规模 production vector workload + 独立 DBMS → Milvus / Qdrant / Weaviate / Vespa
- 闭源 SaaS managed cost-effective object storage → Turbopuffer SPFresh
- 复杂 ranking pipeline + tensor + matryoshka native → Vespa
- 学术 NSG only → 仍零 production OSS DBMS

NSG 在 wiki 内 production OSS DBMS 仍**零部署**——HNSW deployment 6 vendor + SPANN deployment 4 vendor 共同占主流, NSG 仅学术 baseline.

### 已知盲区

- **Chroma OSS Core vs Cloud architectural gap**: 同 codebase 但 ANN algorithm 不同, dev-to-prod migration 路径不公开
- **Chroma Cloud SPANN implementation 细节**: paper [chen-2021-spann] 与 Chroma implementation 差异不公开
- **SPANN 4 production deployment head-to-head**: Bing 历史 + Vespa + Turbopuffer (SPFresh) + Chroma Cloud 实测对比 zero coverage

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/spann.md](../../systems/spann.md)
