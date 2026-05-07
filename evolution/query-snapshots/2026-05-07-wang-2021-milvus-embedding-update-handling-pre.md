---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-07
phase: pre
ingest-context: wang-2021-milvus
wiki-pages-total: 24
cited-pages: [systems/faiss.md, concepts/hnsw.md, concepts/nsg.md]
cited-count: 3
---

# Pre-snapshot: embedding-update-handling

## TL;DR

**Wiki 几乎不覆盖此主题**——所有已 ingest 的 source（HNSW/PQ/NSG/ScaNN/Faiss-GPU/Faiss-survey/DiskANN/SPANN）都假设向量空间稳定。仅 [systems/faiss.md] Open Q 提到"数据分布漂移下的退化"为前沿问题，引用了 Baranchuk 2023 的 explicit updates 工作，但 wiki 未 ingest 该论文。能给出的 wiki 回答仅为"必须全量重建 + 增量 add 受限"。

## Answer

### Wiki 直接覆盖

- [per systems/faiss.md §Open Questions]：
  > "Faiss 当前 IVF / PQ 的 codebook 一旦 train 就冻结，long-running 索引如何 graceful 重新训练？"
  - 引用 Baranchuk 2023 提出 explicit updates，但**该论文 wiki 未 ingest**
  - Faiss 不支持 codebook 增量重训
- [per concepts/hnsw.md §Open Questions]：HNSW 支持 add 不支持 delete / update；图退化是开放问题
- [per concepts/nsg.md §Open Questions]：NSG 完全不支持增量

### 隐含结论

embedding 模型升级（BERT → SBERT 等）意味着：
1. **向量空间几何整体变化**——旧 PQ codebook 与新分布不匹配，**ADC 距离失效**；
2. **图邻居关系失效**——HNSW / NSG 的边在新空间下不再是"邻近"；
3. **分布漂移**——k-means coarse quantizer 中心点偏离，IVF 桶分配错乱。

直接结论：**所有已 ingest 的算法在模型升级时都需要全量重建**。

### Wiki **未覆盖**的工程实践（凭训练知识，标注推测）

> [推测，wiki 未覆盖]
>
> **跨模型兼容/复用方案的常见工业模式**：
>
> 1. **双索引并行 + 流量切换**：旧索引继续服务，新索引后台 rebuild，cutover 时切流量。代价：rebuild 期间内存/SSD 翻倍。
> 2. **CDC + 增量索引 patch**：新模型只对增量数据嵌入；老数据保留旧 embedding，查询端 ensemble 两个索引。代价：检索结果合并复杂、不能做精确 cosine。
> 3. **学习一个 mapping function**：训练 small MLP 把旧 embedding 映射到新 embedding 空间；适用于 fine-tuning 级模型升级，不适用于跨架构。
> 4. **增量 fine-tune 共享空间**：用 contrastive loss 让 v2 embedding 在 v3 空间中也有意义；研究方向，工业未广泛部署。
> 5. **Re-embedding pipeline**：保留原始文档，定期跑 batch re-embed + re-build。是大多数工业系统的实际做法（Pinecone / Milvus / Weaviate 都依赖此）。

> [推测，wiki 未覆盖]
>
> **2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models***（用户提到 talk 当天 live demo）：直接以此问题为主题，提出跨模型整合方案。Wiki 当前**完全未 ingest 该论文**——live demo 后将填补这一最大盲区。

### 已知盲区（trigger 后续 ingest）

- **Baranchuk 2023 explicit updates**（Faiss 论文 §6.1 引用）：codebook 在线更新工程方案
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 live demo 主题，**wiki 当前 zero coverage**
- **Embedding lifecycle 工程模式**：双索引切换、增量 patch、共享空间训练等，都属 zero coverage
- **Milvus / Pinecone 的实际 model upgrade workflow**：DBMS 层的 schema migration / re-embedding pipeline，wiki 未覆盖

## Cited Pages

- [systems/faiss.md](../../systems/faiss.md)
- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
