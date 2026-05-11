---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, systems/distributedann.md, systems/diskann.md]
cited-count: 4
---

# Post-snapshot (adams-2025-distributedann): quantization-landscape

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN 引入 wiki 内首个明示"OPQ + 10× duplication"对换"减 IO + sublinear scaling"的 production trade-off**——为了让 single graph 跨 1000+ machines viable, OPQ 压缩 representation **复制到所有 graph nodes 中** (每 vector 出现在 R=100 个 neighbors 上), 10× space amplification 换 1 IO per node read (vs DiskANN single-node 的 I × R lookup)。这是 quantization landscape 中一个**新维度**: 不仅 quantization 选什么 codec, 还涉及 codec 数据**如何在 distributed system 中布局** (in-place vs duplicated to graph nodes). OPQ 是 Bing production 当前选择 (d_OPQ=64 over d=384, 6× compress)。

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| OPQ production case | Faiss + Milvus IVF_OPQ | **+ DistributedANN at Bing 50B-scale production (d_OPQ=64 over d=384)** |
| Quantization 与 layout 的关系 | layout 是索引细节 | **DistributedANN: OPQ 数据 layout 跨节点是 first-order design** |
| Compressed vector duplication trade | wiki 未明示 | **NEW: 10× space amp 换 1 IO per node** (Eq 1) |
| RaBitQ production | 仍仅 future | **不变** |

### DistributedANN OPQ + duplication trade-off（NEW）

[per adams-2025-distributedann §2.2 Eq 1]

> Our first modification is based on the observation that, for a sufficiently large index, the array of compressed vectors will not be able to fit in a single machine. One option would be to store these compressed vectors in a memory-based key-value store. However, due to the large number of candidates that must be considered (I × R may be tens of thousands of lookups per search for typical parameters) and the latency implications of doing an extra network hop per beam search iteration, we instead decide to **duplicate the compressed representation of each vector into all the graph nodes it is a neighbor of**.

**Eq 1 space amplification**:
```
(1+R) × sizeof(id) + d + R × d_OPQ
─────────────────────────────────── ≈ 10× (R=100, d=384, d_OPQ=64, 8-byte id)
       R × sizeof(id) + d
```

**核心 insight**：每 vector 在 graph 中作为 R=100 个其他 vector 的 neighbor, 它的 OPQ 表示被复制 100+1 次。Storage 10× 但每 graph hop 只需 read 1 graph node 即得 R 邻居的 OPQ 表示 → 减少 lookup roundtrips.

→ Quantization 选择不孤立——**与 distributed layout 联合优化**：
- Single-node DiskANN: PQ in DRAM (compact)
- Distributed DistributedANN: **OPQ duplicated into graph nodes** (10× space, 1 IO per hop)

### Quantization landscape 全景表（updated 2026-05-11 post adams-2025）

| 方法 | 压缩率 | 精度损失 | 工业 production 出现 |
|---|---|---|---|
| Float32 (baseline) | 1× | 0 | 全部 |
| bfloat16 | 2× | 极小 | Vespa first-class |
| Float16 (FP16) | 2× | 极小 | Weaviate + Turbopuffer namespace |
| Scalar Quantization (int8) | 4× | 小 | Qdrant default + Weaviate SQ + Vespa cell + Turbopuffer 哲学 |
| RQ8 (rotation + scalar) | 4× | 比 SQ8 小 | Weaviate default |
| Binary (1-bit) | 32× | 中等 | Qdrant BQ + Weaviate BQ + Vespa single-bit |
| Qdrant 1.5/2-bit | 16-21× | 介于 SQ/BQ | Qdrant only |
| PQ (8x8 = 64-bit) | 24-48× | 中 | Faiss + Milvus + Pinecone + Weaviate option |
| **OPQ (d_OPQ=64 over d=384)** | **6×** | 比 PQ 小 | Faiss + Milvus IVF_PQ + **DistributedANN @ Bing 50B production** |
| RaBitQ | 32× (1-bit) | unbiased + bound | 仅 Faiss + Milvus future |
| QAT model output (int8 → f16) | 8× | 0 (per Voyage QAT) | Turbopuffer |

### Layout 维度（NEW DistributedANN 引入）

| Layout 选择 | 哲学 | 代价 | 优势 |
|---|---|---|---|
| Compact (传统 DiskANN PQ in DRAM) | 全 vector 各占一份 | Memory bound 紧 (3 TiB for 50B × 64 byte) | 最小 storage |
| **Duplicated into graph nodes (DistributedANN)** | 每 vector 复制到 R 个 neighbor 节点 | **10× storage amplification** | **1 IO per graph hop** (vs I × R lookup) |
| KV-stored separately (一个候选) | OPQ 数组独立 KV-stored | 每 hop +1 network roundtrip | Memory 不受限 |
| Shared memory pool (DistributedANN future) | dense cluster fully connected | 需要 specialized HW | 减 storage amp |

### Bing OPQ 配置 vs 其他 production case

- **DistributedANN @ Bing**: d=384 int8, d_OPQ=64 (6× compress), R=72 (truncated from DiskANN R=100), int8 vector + OPQ 复制到 graph node
- **Turbopuffer SPFresh**: 不公开内部 quantization (推测 OPQ-family per LIRE compat)
- **Faiss IVF_OPQ**: standard production usage
- **Milvus**: IVF_PQ / SCANN / DISKANN 选项, OPQ via Faiss option

### 已知盲区

- **DistributedANN 用 OPQ 而非 RaBitQ 原因**：论文未讨论, 可能 (a) 工程 inertia (Bing 已有 OPQ pipeline), (b) RaBitQ 论文 [gao-2024 §4] 明示 graph-based 集成"would require much more efforts"
- **OPQ + duplication 的 cumulative error**: 100 copies are identical (deterministic OPQ encode), 但是否 graph traversal 多次用同一 OPQ representation 累积 estimation error? 论文未讨论
- **R=72 vs DiskANN R=100 recall 影响**: DistributedANN 截 neighbors 节省 storage, 但 recall 影响独立量化未做
- **多 vector 共置 graph node future direction**: §5.1 提"placing multiple nearby full-dimension vectors into a single graph node"未做实验

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/diskann.md](../../systems/diskann.md)
