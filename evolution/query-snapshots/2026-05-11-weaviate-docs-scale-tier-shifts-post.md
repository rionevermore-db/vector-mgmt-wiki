---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 2
---

# Post-snapshot (weaviate-docs): scale-tier-shifts

## TL;DR (delta from qdrant-docs post)

**Weaviate 不引入新质变维度，但**揭示 **8b 维度 (vector-first DBMS) 的子分化深化**——之前 wiki 内 vector-first DBMS 主要 Milvus / Pinecone / Qdrant; Weaviate 推 "**AI-native primary database**" 定位（vector + objects + 完整 inverted index 套件 + 集成 agent stack）—— 是 8b "vector-first" 内部的 **vendor stack vertical integration** 路径。**新增 sub-positioning 8b-i (vendor-stack-integrated vector-first DBMS)** vs 8b-ii (pure vector store + 用户拼上层 agent stack)。但不构成新质变层（部署假设维度未变）。

## Answer

### 质变维度 (unchanged structure, refined 8b)

[per scale-tier-shifts 累积 framework]

| # | 质变点 | 维度 |
|---|---|---|
| 1 | 索引必要性 | 规模 (~10M) |
| 2 | 单机 RAM 触顶 | 规模 + 存储 (~1B) |
| 3 | 单机 SSD 触顶 | 规模 + 存储 (~百亿+) |
| 4 | library → DBMS | 工程形态 |
| 4a | single-server CPU+SSD DBMS | 部署假设 |
| 4b | segment-level Milvus disaggregated | 部署假设 |
| 4c | GPU memory CAGRA | 部署假设 |
| 4d | Rust 单 binary Qdrant | 部署假设 |
| 5 | out-of-place → in-place | update strategy |
| 5a | cluster path SPFresh/LIRE | memory budget ≤10 GB |
| 5b | graph path FreshDiskANN/FreshVamana | memory budget ~128 GB |
| 6 | static index → adaptive | 索引设计 |
| 7 | strong/eventual → tunable τ | consistency |
| **8** | **vector-first DBMS → DB-extended** | **架构起点** |
| **8a** | OLAP-extended (ADBV) | 架构起点 |
| **8b** | OLTP-extended (PASE) | 架构起点 |
| **(8b 之外)** | **vector-first DBMS itself**: Milvus / Pinecone / Qdrant / Weaviate | 架构起点 |
| 9 | search-time filter → build-time filter-aware | filter selectivity |
| 10 | filter-aware build → predicate-agnostic | filter cardinality |
| 11 | TopK interface → Iterator + RM | query 复杂度 |
| 11b | Biased PQ → Unbiased + sharp bound | quantizer 理论保证 |

### Vector-first DBMS 内的"AI-native vs Tool-box"分化（NEW）

[per systems/weaviate.md "与 Milvus 的对比" + systems/qdrant.md "与 Milvus 的对比"]

之前 wiki 把 vector-first DBMS 视为 monolithic group；Weaviate ingest 暴露 **vendor stack 哲学**子分化：

| Vendor stack 哲学 | 代表 |
|---|---|
| **Tool box for all workloads** (多 index_type) | Milvus (HNSW/IVF*/DISKANN/CAGRA/SPARSE) |
| **Do HNSW exceptionally well** (Rust + 简洁 + filter-aware) | Qdrant |
| **AI-native primary DB** (vector + objects + inverted indexes + agent stack vertical) | **Weaviate** |
| **Managed simplicity SaaS** | Pinecone (闭源) |

→ **vector-first DBMS 内部至少 4 种 vendor 哲学**——但都是同 8 维度 (vector-first 起点)，不构成质变层。

### "AI-native vendor stack" 是否构成新质变？

**不构成** —— Weaviate Query Agent + Engram 是 vendor 上层产品 (RAG turnkey + agent memory)，**不改变 DBMS 自身 architecture**。Pinecone 也有 Pinecone Inference，Milvus 通过 Zilliz Cloud 提供管理服务。**Vendor stack 厚度** 是商业差异化而非技术质变。

但 vendor stack 影响 production 选择：RAG-heavy app 选 Weaviate / Pinecone；自管 LLM stack 选 Milvus / Qdrant。

### 与之前 ingest 的累积演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| 质变维度数 | 12 (with 4a/4b/4c/4d + 5a/5b + 11b) | **12 (不变)** |
| Vector-first DBMS 子分化 | + Qdrant Rust 简洁 (4d) | **+ Weaviate AI-native + agent stack** (vendor 哲学层) |
| OSS production engineering 路径 | 4 种 (Milvus / Qdrant / Faiss-library / Weaviate-pending) | **完整 4 种 OSS** (Milvus / Qdrant / Weaviate + Faiss-library) |

### 不算质变（参数微调）

- Weaviate RQ8 vs RQ1 配置
- Weaviate hybrid alpha (BM25/vector 融合权重)
- Weaviate sharding / replication_factor
- Weaviate Query Agent / Engram 启用

### 已知盲区

- **Weaviate AI-native production 实证**：完全空白
- **维度 4-equivalent + Weaviate Go production engineering 路径子分化** (与 Qdrant 4d Rust 形成对应)：未独立标识
- **维度 12+**：未来 ingest 是否暴露更多

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
