---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/vamana.md, systems/milvus.md, systems/diskann.md, topics/index-selection.md]
cited-count: 6
---

# Post-snapshot (milvus-docs): hnsw-vs-nsg-selection

## TL;DR (delta from wang-2021-milvus post)

**RNSG 身份歧义部分缓解**：v2.6.x docs 把 **DISKANN 作为独立索引列出**（[per sources/docs/milvus/site/en/about/limitations.md] 索引矩阵）——DISKANN 与 RNSG 是两个不同选项。这反向支持 RNSG 是 [NSG](../../concepts/nsg.md) 工程实现而非 Rand-NSG（[Vamana](../../concepts/vamana.md)）。**新增 v2.6.x 选型考虑**：HNSW 与 SCANN（Google ScaNN）、GPU_CAGRA 同时存在，Milvus 工业部署的 graph 选项现在有 4-5 种。

## Answer

### v2.6.x 文档对 RNSG 身份的间接证据 [per systems/milvus.md "索引家族"]

v2.6.x 索引列表 [per sources/docs/milvus/site/en/about/limitations.md]：

```
HNSW | DISKANN | FLAT | IVF_FLAT | IVF_SQ8 | IVF_PQ | SCANN |
GPU_IVF_FLAT | GPU_IVF_PQ | GPU_CAGRA | GPU_BRUTE_FORCE |
SPARSE_INVERTED_INDEX | BIN_FLAT | BIN_IVF_FLAT
```

**没有 RNSG**——v2.6.x 文档**不再列 RNSG**！

可能的解释：
1. RNSG 在 v2.x 中被改名（如改为不同 graph 名）
2. RNSG 被合并/重写（与 HNSW 或 DISKANN）
3. v2.6.x 只列"主流推荐"而非全部支持的索引

无论原因，v2.6.x DISKANN 独立列出意味着 1.x 的 RNSG（如果是 Rand-NSG）已被 DISKANN 取代，与 HNSW 是两个独立 graph 选项。

### v2.6.x graph 选型决策树（NEW）

| 场景 | 推荐 | 备注 |
|---|---|---|
| 全内存 + 高 recall | HNSW | v2.6.x 默认 graph |
| 内存预算极小 + SSD 充足 | DISKANN | v2.6.x 集成 [Vamana](../../concepts/vamana.md) |
| GPU 节点可用 + throughput 优先 | GPU_CAGRA | NVIDIA RAFT |
| MIPS 任务 + 高 recall | SCANN | Google ScaNN，anisotropic |
| 稀疏向量 / 全文搜索 | SPARSE_INVERTED_INDEX | BM25 + SPLADE / BGE-M3 |

**关键 take**：v2.6.x 把 graph 选型从"HNSW vs NSG vs FANNG"二三选一扩展为"HNSW vs DISKANN vs CAGRA vs SCANN" 多模态选型——选择维度不再只是 million-scale QPS，还包括存储介质、硬件、相似度类型。

### 与 wang-2021 post 的差异

| | wang-2021 post（1.x） | milvus-docs post（2.x） |
|---|---|---|
| Milvus graph 索引 | HNSW + RNSG（身份歧义） | HNSW + DISKANN + GPU_CAGRA + SCANN |
| 选型基础 | million-scale QPS | + 存储介质 + 硬件 + 相似度类型 |
| 1.x SQ8H 自研索引 | 自研 hybrid CPU/GPU | **v2.6.x 文档未列**——可能被 GPU_CAGRA 取代 |
| HNSW 在 Milvus 中的地位 | 默认 graph | 仍是默认，但有更多专用替代 |

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/index-selection.md](../../topics/index-selection.md)
