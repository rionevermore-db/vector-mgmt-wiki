---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/weaviate.md, systems/qdrant.md]
cited-count: 4
---

# Post-snapshot (weaviate-docs): hnsw-vs-nsg-selection

## TL;DR (delta from qdrant-docs post)

**Weaviate ingest 进一步强化"HNSW 是 production OSS vector DBMS 主流" landscape**——Weaviate 与 Qdrant 都 only HNSW（无 NSG/IVF/Flat）。**关键 NEW**：Weaviate **HFresh (preview)** 是 wiki 内首个**HNSW base streaming/freshness 工程化项目**——之前 graph-path streaming 仅 Vamana base ([FreshVamana](../../concepts/freshvamana.md))。HFresh + FreshVamana 形成 graph-path streaming 双路径：HNSW base (Weaviate) vs Vamana base (FreshDiskANN / Microsoft)。

## Answer

### 与之前 ingest 的演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | Qdrant only-HNSW + Milvus 多 | **+ Weaviate only-HNSW + RQ8 default** |
| NSG production 出现 | 仍仅 Taobao 2B | **不变** |
| HNSW streaming/freshness 项目 | 仅 [FreshVamana](../../concepts/freshvamana.md) Vamana base | **+ Weaviate HFresh preview (HNSW base!)** |
| HNSW + filter-aware build | Qdrant Filterable HNSW + ACORN | **+ Weaviate ACORN integration + positive/negative correlation optimization** |

### HFresh - 新的 HNSW streaming branch（NEW）

[per sources/docs/weaviate/llms.txt §Architecture]

> **HFresh (in preview)** may become the default; it offers better freshness guarantees for frequently updated data.

→ 这是 wiki 内**首个 HNSW base 的 streaming graph 项目**。之前 graph-path streaming 路径全部 Vamana base（[FreshVamana](../../concepts/freshvamana.md) + α-RNG property）。HFresh 算法细节 docs 未公开——可能：
- HNSW + α-augmented RobustPrune (理论上 hnswlib 等价 patch logical)
- OR HNSW + 其他 streaming-friendly modification
- OR 全新算法

**实证未在 docs**——但 Weaviate 明示 HFresh 是 "**may become the default**"——production-ready 投入。

### HNSW vs NSG 选择决策表（updated）

| 工程考量 | HNSW | NSG |
|---|---|---|
| Production engineering 深度 | **Milvus + Qdrant + Pinecone + Faiss + VBASE + Weaviate 全覆盖** | NSSG + Taobao only |
| Filter-aware build | ✓ via ACORN / Qdrant Filterable HNSW / Weaviate ACORN | ✗ |
| Streaming insert+delete | **HNSW α=1 fail** → 需 α=1.2 patch (FreshVamana on Vamana base; **Weaviate HFresh** preview on HNSW base) | ✗ |
| GPU-native | ✓ via CAGRA | ✗ |
| OSS DBMS 集成 | **Milvus / Qdrant / Weaviate 三大 OSS 都选 HNSW** | NSSG (academic) only |
| Disk-resident segment | ✓ via Starling-HNSW | ✓ via Starling-NSG |
| Iterator + RM | ✓ via VBASE | ✗ |
| Streaming graph 在 production | **HFresh (HNSW base, Weaviate preview)** + FreshVamana (Vamana base) | n/a |

### 选择决策（updated 2026-05-11）

- **CPU + 简单 single-vector TopK + million scale + 静态 + 内存富余** → NSG
- **OSS vector DBMS production (主流)** → **HNSW** (Qdrant / Milvus / Weaviate / Faiss)
- **OSS Rust + 简洁 + filter heavy** → Qdrant
- **OSS Go + AI-native primary DB + agent stack** → **Weaviate**
- **OSS Go + 多 index_type + cloud-native** → Milvus
- **闭源 SaaS** → Pinecone
- **Streaming + graph path** → Vamana base (FreshDiskANN) OR **HNSW base (Weaviate HFresh preview)**

NSG 仍是"专攻 single-vector TopK 学术 benchmark"——production vector DBMS 主流仍是 HNSW。

### 已知盲区

- **HFresh 算法细节**：preview 未公开实现
- **HFresh vs FreshVamana 性能对比**：两个 graph-path streaming 实现的算法/性能比较 wiki 内 zero coverage
- **NSG-based 主流 production DBMS**：仍未出现
- **HNSW + ACORN + HFresh + RQ8 联合性能**：Weaviate 默认 stack 实测 wiki 未量化

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/qdrant.md](../../systems/qdrant.md)
