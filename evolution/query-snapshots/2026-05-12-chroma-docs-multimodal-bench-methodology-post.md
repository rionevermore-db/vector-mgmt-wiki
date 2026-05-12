---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, topics/multimodal-embedding-retrieval.md]
cited-count: 2
---

# Post-snapshot (chroma-docs): multimodal-bench-methodology

## TL;DR (post-ingest of chroma-docs)

**Chroma 多模态 / spatial 能力**与其他 vector DBMS 一致——**dense vector cosine ANN native 但 spatial 仅 attribute filter**. **关键 NEW**: Chroma 添加 **MCP server first-class** 让 multimodal benchmark 包含 **AI-agent-context retrieval workload** 维度——Package Search MCP 是 wiki 内**首个把 source-code retrieval 作 production case** 的 vendor (12+ AI platforms Claude / Cursor / Codeium 等). 这是 multimodal 之外的**新 retrieval workload category**——但 spatial 维度仍 wiki industry gap.

## Answer

### Wiki 9 vendor 空间能力 inventory（updated 2026-05-12 post chroma-docs）

| Vendor | 真 spatial 路径 |
|---|---|
| Milvus | ✗ |
| Qdrant | 半 native (geohash filter) |
| Weaviate | ✗ |
| Pinecone | 不公开 |
| **Vespa** | **✓ native** (position dimension) |
| Turbopuffer | ✗ |
| DistributedANN | paper 不涉及 |
| pgvector | **✓ via PostGIS ecosystem** |
| **Chroma (NEW)** | **✗ (无 native spatial; 仅 metadata filter)** |

→ wiki 内 native spatial vendor 仍 2 个 (Vespa + pgvector + PostGIS combo). Chroma 不增加 spatial native.

### Chroma AI-agent retrieval workload category (NEW)

[per sources/docs/chroma/llms-full.txt §Package Search MCP Server]

Chroma 通过 **MCP server first-class** 添加新 retrieval workload category:
- AI agents (Claude / Cursor / Codeium / 等 12+) 通过 MCP 协议调用 Chroma
- Package Search MCP: retrieve source code from npm / PyPI / crates.io / Go modules / GitHub
- AI agent context augmentation: 减少 LLM hallucination 通过 ground-truth code retrieval

→ **这是 wiki 内 first vendor 把 "AI agent retrieval workload" 作 production-shipped first-class feature**. Weaviate Query Agent 类似但 vendor-bundle, Chroma MCP 是 protocol-level.

### Multimodal benchmark methodology (updated 2026-05-12 post chroma-docs)

3 axis benchmark requirement (sparse + dense + spatial + filter) 之外, **应增加 axis**: **AI-agent workload integration** (vendor 是否原生支持 AI agent retrieval via standard protocol).

| Vendor | AI agent protocol native |
|---|---|
| Weaviate | Query Agent (vendor-bundle, not protocol) |
| Pinecone | Pinecone Assistant (vendor-bundle, not protocol) |
| **Chroma** | **MCP server first-class (Anthropic protocol native, 12+ AI platforms)** |
| Others | application-level integration |

### 已知盲区

- **Chroma MCP server vs Weaviate Query Agent production case 对比**: 不公开
- **MCP-based retrieval workload benchmark methodology**: 不存在公开 standard
- **Chroma + spatial via PostGIS-style ecosystem**: 不存在 (Chroma 不基于 Postgres)

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
