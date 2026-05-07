---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [concepts/product-quantization.md, concepts/scann.md, systems/milvus.md, systems/faiss.md, topics/index-selection.md]
cited-count: 5
---

# Post-snapshot (milvus-docs): quantization-landscape

## TL;DR (delta from wang-2021-milvus post)

**v2.6.x 索引家族大幅扩展**：[per systems/milvus.md "v2.6.x 索引家族"]
- 新增 **DISKANN**（PQ DRAM + SSD 全精度，[Vamana](../../concepts/vamana.md) 算法移植）
- 新增 **SCANN**（[Google ScaNN](../../concepts/scann.md) anisotropic PQ，wiki 内首次工业系统集成）
- 新增 **GPU_CAGRA**（NVIDIA RAFT，graph + GPU）
- 新增 **SPARSE_INVERTED_INDEX**（BM25 + SPLADE / BGE-M3 学习稀疏 embedding）
- **SQ8H（1.x 自研）从 v2.6.x 文档消失**——被 GPU_CAGRA 等取代

## Answer

### v2.6.x 量化/压缩索引完整列表（NEW）

[per sources/docs/milvus/site/en/about/limitations.md, about/overview.md]：

| 类别 | 索引 | 量化方法 | 备注 |
|---|---|---|---|
| 无量化 | FLAT, IVF_FLAT | 无 | 全精度 |
| Scalar quantization | IVF_SQ8 | SQ8 | 同 1.x |
| Product quantization | IVF_PQ | PQ | 同 1.x |
| **Score-aware PQ** | **SCANN** | **anisotropic PQ** | NEW；集成 [Google ScaNN](../../concepts/scann.md) |
| **Disk + 量化导航** | **DISKANN** | PQ DRAM + SSD 全精度 | NEW；集成 [Vamana](../../concepts/vamana.md)/[DiskANN](../../systems/diskann.md) |
| GPU 量化 | GPU_IVF_FLAT, GPU_IVF_PQ | 无 / PQ | NVIDIA |
| **GPU graph** | **GPU_CAGRA** | 无 | NEW；NVIDIA RAFT |
| GPU brute force | GPU_BRUTE_FORCE | 无 | NEW |
| 稀疏 | SPARSE_INVERTED_INDEX | 无 | NEW；BM25 / SPLADE |
| 二进制 | BIN_FLAT, BIN_IVF_FLAT | 二进制 | Hamming |

### SCANN 在 Milvus 的集成意义（NEW）

[per concepts/scann.md, systems/milvus.md] 之前 wiki 已有 ScaNN page，但只描述算法本身。**v2.6.x 是 wiki 内首次记录 ScaNN 被工业 vector DBMS 直接集成**——不仅 Faiss 借鉴 4-bit FastScan SIMD layout，Milvus 把整个 SCANN 索引作为选项暴露给用户。

工程含义：MIPS 任务（推荐系统、softmax 近似）下用户可在 Milvus 直接选 `index_type=SCANN`，无需切换到独立 Google ScaNN 服务。

### DISKANN 在 Milvus 的集成意义（NEW）

[per systems/diskann.md, systems/milvus.md] DiskANN 算法（Vamana + PQ + SSD）原本是 Microsoft 独立开源系统，与 Milvus 是平行竞品。v2.6.x **把 DiskANN 作为索引选项纳入 Milvus 索引族**——用户可在 Milvus collection 创建时选 `index_type=DISKANN` 享受 SSD-resident + 高 recall 的能力。

> **wiki 解读**：1.x 时代 Milvus = "DBMS + Faiss-only 算法核心"；2.x 时代 Milvus = "DBMS + 多算法核心（Faiss + DiskANN + ScaNN + RAFT）"。DBMS 与 algorithm 的边界进一步松动。

### SQ8H 的消失（observation）

[per systems/milvus.md] 1.x SIGMOD 论文的 SQ8H（hybrid CPU/GPU SQ8 索引）是 §3.4 重点贡献。v2.6.x 文档**未列 SQ8H**。可能解释：
- GPU_CAGRA 解决了 SQ8H 试图解决的问题（GPU 装不下数据时的性能优化）
- NVIDIA RAFT 生态成熟后，SQ8H 自研路线让位

### Open / 仍未覆盖

- **OPQ 独立 page**：v2.6.x 仍不直接列 OPQ
- **RaBitQ**：完全未覆盖（Milvus v2.6.x 也未集成）
- **SCANN 在 Milvus 实测数字**：wiki 现有 ScaNN benchmark 是 Glove1.2M（Google 自家），Milvus 集成后的数字未覆盖
- **GPU_CAGRA 与 SQ8H 性能直接对比**：1.x → 2.x 切换的工程理由文档未深入

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/scann.md](../../concepts/scann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/faiss.md](../../systems/faiss.md)
- [topics/index-selection.md](../../topics/index-selection.md)
