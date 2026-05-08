---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: pinecone-docs
wiki-pages-total: 36
cited-pages: [systems/pinecone.md, systems/milvus.md, concepts/pinecone-serverless-slabs.md]
cited-count: 3
---

# Post-snapshot (pinecone-docs): embedding-update-handling

## TL;DR (delta from xu-2023-spfresh post)

**Pinecone 与所有 wiki 已 ingest source 一样不解决根本问题**：embedding model 升级仍需新 index + 双索引切换。Pinecone 提供更多 hosted embedding model（一并 cutover 时方便）但**算法 / 协议层无新方案**。**Index 一旦创建 dimensionality 不可改**——切 model 等于建新 index。

## Answer

### Pinecone 在 embedding lifecycle 上的现状（NEW）

[per systems/pinecone.md "数据模型 + Open Q"]

**支持**：
- ✓ Hosted embedding models（"integrated embedding"）：upsert 时传 raw text，Pinecone 自动 embed
- ✓ External embedding models：用户自己 embed
- ✓ 多 index 共存（同 project 不同 model 不同 index）
- ✓ Backup（snapshot index 用于恢复）

**不支持**：
- ✗ 单 index 内切换 embedding model（dimension / metric 一旦定不能改）
- ✗ 跨 model 自动 mapping function
- ✗ Online migration 单 namespace 切到新 embedding 空间

→ 与 [Milvus](../../systems/milvus.md) 同——基础设施提供，**模型升级 playbook 仍需用户实现**（双 index + 流量切换）。

### Pinecone 工具链支持的双索引切换（NEW）

[per systems/pinecone.md "数据模型 + 生产案例"]

Pinecone 提供的 cutover 友好特性：
1. **Backup → 新 index restore**（不能跨 dimension）：仅适合同 model 升级（v2 → v2.1）的恢复场景
2. **Pinecone Inference (hosted models)**：embedding 与 index 同一服务方便 batch re-embed
3. **CLI / API automation**：脚本化 batch upsert
4. **Namespace-level migration**：可在同 index 多 namespace 测试新数据，但仍需 dimension 相同

> **限制**：与 Milvus 的 CDC 工具不同，Pinecone 不提供原生 incremental sync 工具——双索引并行期间增量同步必须**应用层自实现**。

### 与之前 ingest post 的差异

| | wang-2021 post | milvus-docs post | xu-2023-spfresh post | **pinecone-docs post (NEW)** |
|---|---|---|---|---|
| Vector update 解 | LSM | + CDC + function field | + in-place LIRE | + **slab adaptive merge** |
| Embedding upgrade 解 | 双索引（手动） | + CDC | 仍 zero algorithmic | 仍 zero algorithmic（**SaaS 也无方案**） |
| Hosted embedding model | n/a | function field | n/a | **Pinecone Inference 集成** |

### 已知盲区（仍未覆盖）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo，仍未 ingest，**唯一已知有 algorithmic 答案的 source**
- **Pinecone 实际 embedding 升级工作流**：blog / community 可能有最佳实践但未 ingest
- **跨 model embedding mapping function 训练**：研究方向 zero coverage
- **共享 embedding space contrastive fine-tune**：研究方向 zero coverage

## Cited Pages

- [systems/pinecone.md](../../systems/pinecone.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/pinecone-serverless-slabs.md](../../concepts/pinecone-serverless-slabs.md)
