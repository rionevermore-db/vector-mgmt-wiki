---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [systems/faiss.md, systems/milvus.md, concepts/hnsw.md, concepts/nsg.md]
cited-count: 4
---

# Post-snapshot: embedding-update-handling

## TL;DR

**Wiki 仍几乎不直接覆盖此主题**——所有已 ingest 的 source 都假设向量空间稳定。**新增 Milvus 的 LSM 视角**：即使 Milvus 支持 dynamic data（持续 insert/delete + tiered merge），论文 §2.3 的"动态数据"指**已有 embedding 空间内的增删改**，不指**embedding 空间本身切换**。模型升级在 Milvus 当前架构下仍需 schema migration + re-embedding pipeline，**论文未直接处理**。

## Answer

### Wiki 直接覆盖（updated）

- [per systems/faiss.md §Open Questions]：
  > "Faiss 当前 IVF / PQ 的 codebook 一旦 train 就冻结，long-running 索引如何 graceful 重新训练？"
  - 引用 Baranchuk 2023 explicit updates，但**该论文 wiki 未 ingest**
- [per concepts/hnsw.md §Open Questions]：HNSW 支持 add 不支持 delete / update
- [per concepts/nsg.md §Open Questions]：NSG 完全不支持增量
- [per systems/milvus.md §Open Questions]（**新增**）：
  > "Embedding model 升级处理：所有动态数据假设向量空间稳定。模型升级（BERT→SBERT）下的 schema migration / re-embedding pipeline 论文未讨论"
- [per systems/milvus.md §"动态数据 (LSM 模式)"]：Milvus 的 LSM 是**已有 embedding 空间内**的 insert/delete + segment merge；模型升级是**整个 embedding 空间切换**——Milvus 论文未直接覆盖这一情况

### 隐含结论（仍然成立）

embedding 模型升级（BERT → SBERT 等）意味着：
1. **向量空间几何整体变化**——旧 PQ codebook 与新分布不匹配，**ADC 距离失效**
2. **图邻居关系失效**——HNSW / NSG / RNSG 的边在新空间下不再是"邻近"
3. **分布漂移**——k-means coarse quantizer 中心点偏离，IVF 桶分配错乱
4. **Milvus segment-level index 也失效**——每 segment 内 index 都是按旧 embedding 训的

**结论**：所有已 ingest 的算法/系统在模型升级时都需要全量重建，**Milvus 的 dynamic data 机制不能解决此问题**——它解决的是"同一空间内数据增删"，不是"空间切换"。

### Milvus 在模型升级场景的可能用法（推断）

> [推测，wiki 未覆盖具体工业实践]
>
> 利用 Milvus 现有特性可以**实现**双索引切换模式：
>
> 1. **创建两个 Milvus collection**（旧空间 + 新空间）
> 2. 后台批量 re-embed 数据写入新 collection（用 LSM 模式承接持续写入）
> 3. Snapshot isolation 保证旧 collection 服务不被新 collection 写入影响
> 4. 流量切换：query 路由器从旧切到新
>
> 这是把 Milvus 当工具用，不是 Milvus 论文的论点。

### Wiki **未覆盖**的工程实践（凭训练知识，标注推测）

> [推测，wiki 未覆盖]
>
> **跨模型兼容/复用方案的常见工业模式**（与 pre-snapshot 同）：
>
> 1. **双索引并行 + 流量切换**：旧索引继续服务，新索引后台 rebuild，cutover 时切流量。代价：rebuild 期间内存/SSD 翻倍。**Milvus 的 multi-collection + snapshot isolation 是此模式的天然支持**
> 2. **CDC + 增量索引 patch**：新模型只对增量数据嵌入；老数据保留旧 embedding，查询端 ensemble。代价：检索结果合并复杂
> 3. **学习一个 mapping function**：训练 small MLP 把旧 embedding 映射到新 embedding 空间
> 4. **增量 fine-tune 共享空间**：用 contrastive loss 让 v2 embedding 在 v3 空间中也有意义
> 5. **Re-embedding pipeline**：保留原始文档，定期跑 batch re-embed + re-build——**大多数工业系统的实际做法**

> [推测，wiki 未覆盖]
>
> **2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models***（用户 talk 当日 demo）：直接以此问题为主题。Wiki 当前**完全未 ingest 该论文**——live demo 后将填补这一最大盲区。

### 已知盲区（trigger 后续 ingest）

- **Baranchuk 2023 explicit updates**（Faiss 论文 §6.1 引用）：codebook 在线更新工程方案
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 live demo 主题，**wiki 当前 zero coverage**
- **Embedding lifecycle 工程模式**：双索引切换、增量 patch、共享空间训练等，仍 zero coverage（虽 Milvus 提供基础能力但论文未讨论应用）
- **Milvus 2.0+ schema migration**：cloud-native 重写后是否原生支持 collection 间 re-embedding？wiki 未覆盖

## Cited Pages

- [systems/faiss.md](../../systems/faiss.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
