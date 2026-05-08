---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: pinecone-docs
wiki-pages-total: 36
cited-pages: [concepts/product-quantization.md, systems/pinecone.md, concepts/pinecone-serverless-slabs.md, topics/index-selection.md]
cited-count: 4
---

# Post-snapshot (pinecone-docs): quantization-landscape

## TL;DR (delta from xu-2023-spfresh post)

**Pinecone 是 wiki 内首个 quantizer 完全黑盒的系统**：用户不能选 PQ / OPQ / SQ；Pinecone serverless 内部的 adaptive indexing 可能在不同 slab 用不同 quantizer 但**完全不公开**。这与 [Milvus v2.6.x](../../systems/milvus.md) 显式列 IVF_FLAT / IVF_SQ8 / IVF_PQ / SCANN 形成对比。

## Answer

### Pinecone quantization 黑盒（NEW）

[per systems/pinecone.md "Index type 不暴露用户"]

Pinecone serverless 用户控制范围：
- ✓ Vector field 类型（dense_vector / sparse_vector）
- ✓ Vector dimensionality
- ✓ 相似度 metric（cosine / euclidean / dotproduct）
- ✗ Quantization 算法（PQ / SQ / 任何 variant）
- ✗ Quantization 参数（m, k*, code length）
- ✗ Index 算法（HNSW / IVF / Vamana / ...）

> **wiki 解读**：Pinecone 把 quantizer 完全藏起来——这意味着用户无法做"小 code 换内存 + 高 recall"trade-off 决策。Pinecone 自己判断；用户接受。

### Pinecone Pod-based 时代的隐含 quantization（legacy）

[per concepts/pinecone-pod-based.md "Pod 类型与容量"]

Legacy pod-based 索引隐含 quantization 选择：
- **s1**（storage-optimized, 5M vec/768d/pod）：可能内部用更激进的量化（推断 SQ8 类）
- **p1**（performance, 1M vec/768d/pod）：较少量化，全/接近全精度
- **p2**（high QPS）：不同 trade-off，且**不支持 sparse vector**——意味着 p2 内部 quantizer 与 sparse 不兼容

**数字推算**：768-d × 4B = 3 KB/vec；s1 单 pod 5M 向量 ≈ 15 GB——对应"~32 byte/vec"量化（接近 PQ 8-byte 段或 SQ8）。p1 单 pod 1M ≈ 3 GB——可能不量化或 SQ8。具体未公开。

### Adaptive Indexing 的 quantization 含义（NEW）

[per concepts/pinecone-serverless-slabs.md "Layer 2：Slab"]

Pinecone serverless 推断（**docs 未明示**）：
- **小 slab**（新 flush 内存量化轻）：可能 fast 索引 + 全精度或轻量化
- **大 slab**（merge 后驻留时间长）：可能 sophisticated 索引 + 更激进量化（amortize 量化训练成本）

**意味着同一 namespace 内 quantization 强度按 slab 大小变化**——wiki 内独例。

### 与 wiki 其他 quantizer 路线的对比（updated）

| 系统 | Quantizer 用户可见性 | Adaptive across lifecycle |
|---|---|---|
| Faiss | ✓ 显式选（OPQ16,IVF65536_HNSW32,PQ8x4fs 等） | ✗ |
| Milvus | ✓ index_type 选（IVF_SQ8 / IVF_PQ / SCANN / DISKANN） | ✗（segment 内固定） |
| DiskANN | n/a（PQ default） | ✗ |
| SPANN / SPFresh | n/a（全精度） | ✗ |
| **Pinecone Pod-based** | **隐含**（pod 类型决定） | ✗（pod 选定后冻结） |
| **Pinecone Serverless** | **完全不可见** | **✓（slab merge 自适应）** |

### Open / 未覆盖

- **Pinecone 实际 quantizer**：是 PQ / OPQ / SQ / RaBitQ / 自研——完全不公开
- **Adaptive quantization 性能数据**：与 fixed quantizer 对比 docs 未给
- **Pinecone DRN 的 t1 vs b1 quantization 差异**：t1 缓存"vector index + projections"，可能含全精度向量做 re-rank（类 DiskANN 思路），但具体不公开
- **RaBitQ**：仍未覆盖（学术 + Pinecone 都不直接命名）

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/pinecone-serverless-slabs.md](../../concepts/pinecone-serverless-slabs.md)
- [topics/index-selection.md](../../topics/index-selection.md)
