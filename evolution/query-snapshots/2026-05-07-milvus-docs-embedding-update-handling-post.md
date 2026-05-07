---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [systems/milvus.md, concepts/woodpecker.md]
cited-count: 2
---

# Post-snapshot (milvus-docs): embedding-update-handling

## TL;DR (delta from wang-2021-milvus post)

**v2.6.x 提供更具体的 embedding lifecycle 工具但仍不解决根本问题**：CDC（Change Data Capture）+ embedding 函数内嵌 + 函数字段（function-based field）让"双索引切换"更容易，但 wiki 现有 source 仍**未直接讨论模型升级时的语义保留方案**。Milvus 提供的是**基础设施**，不是**算法解**。

## Answer

### v2.6.x 新增的相关工具（NEW）

[per systems/milvus.md, sources/docs/milvus/site/en/about/overview.md]：

**1. CDC（Change Data Capture）** —— "Milvus-CDC can capture and synchronize incremental data in Milvus instances"
- Primary/standby 灾备模式
- 增量复制：旧→新 collection
- 适用场景：模型升级时，新 collection 持续接收 CDC 同步的增量

**2. Embedding 函数内嵌**：
- PyMilvus 集成主流 embedding model（OpenAI / Gemini / SPLADE / BGE-M3）
- Collection schema 可定义 **function-based field**——文本字段经 embedding 函数自动转 vector，**不需要应用代码处理 embedding**
- 模型升级时改 function 配置即可触发新 vector 生成

**3. Multi-tenancy 隔离**：
- 同一 cluster 内多 collection 隔离 embedding 空间
- 不同 collection 用不同 embedding model 互不影响

### 仍然不能直接解决的问题

[per systems/milvus.md "Open Questions"]：

> "Embedding model 升级处理：所有动态数据假设向量空间稳定。模型升级（BERT→SBERT）下的 schema migration / re-embedding pipeline 论文未讨论"

v2.6.x docs 在用户指南层有 collection 创建/删除/数据 import 文档，但**没有专门的"模型升级 playbook"**——这一缺口 v2.6.x 与 SIGMOD 1.x 同样存在。

### 用 Milvus v2.6.x 工具搭"模型升级"工作流（推断）

> [推测，wiki 文档未直接覆盖]
>
> **跨 collection re-embedding pipeline**：
>
> 1. 创建 collection_v2（用新 embedding model）
> 2. 配置 function-based field 用新 model
> 3. 启动 batch job 把 collection_v1 数据 re-embed 写到 collection_v2
> 4. 启用 CDC 把 collection_v1 增量同步到 collection_v2
> 5. 切流量（Access Layer route 改）
> 6. 删除 collection_v1
>
> v2.6.x 工具集都齐——CDC、function field、多 collection——但 **playbook 是工程实现，不是 Milvus 论点**。

### 与 wang-2021 post 的差异

| | wang-2021 post（1.x） | milvus-docs post（2.x） |
|---|---|---|
| 增量数据机制 | LSM segment + tiered merge | + CDC + function-based field |
| 双 collection 切换 | 可手动实现 | 工具更齐（CDC 现成） |
| 模型升级"playbook" | wiki 完全未覆盖 | wiki 仍未覆盖 |
| 自动 re-embedding | 无 | function field 部分支持但需手动重建 |

### 已知盲区（仍未覆盖，trigger 后续 ingest）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**（talk 当日 demo 主题）：直接以此为主题；wiki 当前 zero coverage
- **Baranchuk 2023 explicit updates**：仍未 ingest
- **Milvus 实际工业 model upgrade 文档**：blog / community 可能有但未 ingest
- **跨 model embedding 兼容性研究**：研究方向，wiki zero coverage

## Cited Pages

- [systems/milvus.md](../../systems/milvus.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
