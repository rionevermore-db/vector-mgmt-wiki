---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, concepts/scann.md, concepts/vgpq.md, benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]
cited-count: 5
---

# Post-snapshot (gao-2024-rabitq): quantization-landscape

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ ingest 是 quantization-landscape query 自 wiki 启动以来的最大单次 delta**——首次提供 wiki 内 unbiased + sharp probabilistic error bound 的 quantizer。**关闭 query 标题中"RaBitQ"4 字以来一直存在的 wiki coverage gap**——question 里直接点名的 4 种方法终于全 cover（PQ / OPQ via Faiss / SQ via Faiss / **RaBitQ now**）。Quantization landscape 不再是 PQ-family 演化路径（PQ → OPQ → ScaNN → VGPQ）单线——**RaBitQ 跳出 PQ 框架开辟"几何 + 理论保证"的新路线**。query 现在能给出**有理论 backing 的精度-速度-内存权衡**而非只 empirical heuristic。

## Answer

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| 新增 quantizer | 无 | **RaBitQ（首个 unbiased + error bound）** |
| Quantization 维度 | vector + graph + lossy（PQ-family） | **+ 理论保证维度**（biased + heuristic vs unbiased + bound） |
| Quantization 与 query interface 关系 | lossy quantizer 与 RM iterator 兼容性开放 | **同样开放**（RaBitQ 是 unbiased，与 RM 更兼容） |
| Wiki 内 quantizer count | PQ / OPQ (via Faiss) / SQ8 / RQ / LSQ / VGPQ / ACORN compression / ScaNN | **+ RaBitQ** |

### Quantization landscape 完整版（updated）

| 方法 | 类型 | code 长度 | error bound | 主要应用 | wiki coverage |
|---|---|---|---|---|---|
| Binary | scalar quantization | D bits | 无 | Faiss BIN_FLAT | indirect |
| SQ8 | scalar quantization | 8D bits | 无 | Faiss / Milvus | indirect |
| PQ | product quantization | M·k bits (default 2D) | **无** | Faiss / Milvus / DiskANN | concepts/pq.md |
| OPQ | rotated PQ | 同 PQ | 无 | Faiss | indirect via Faiss survey |
| RQ / LSQ | additive quantization | 同 PQ | 无 | Faiss / Manu | indirect |
| ScaNN anisotropic | score-aware PQ | 同 PQ | 无 | ScaNN / Milvus | concepts/scann.md |
| VGPQ | PQ + Voronoi 几何剪枝 | 同 PQ | 无 | AnalyticDB-V | concepts/vgpq.md |
| ACORN compression | neighbor list truncation | graph edges | 无 | ACORN-γ | concepts/acorn.md |
| **RaBitQ (NEW)** | **几何 + 随机正交矩阵** | **D bits（一半 PQ 默认）** | **O(1/√D) w.h.p. sharp** | **research; pending integration** | **concepts/rabitq.md** |

→ RaBitQ 是 wiki 内**唯一带 sharp error bound 的 quantizer**——理论上下界（[Alon-Klartag 2017]）证明 D-bit 短码无法更紧，所以 **asymptotically optimal**。

### Quantization-vs-Recall 实测（NEW）

[per benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md]

6 dataset (MSong / SIFT / DEEP / Word2Vec / GIST / Image) 实测 average rel error：

| 方法 | SIFT D=128 | GIST D=960 | **MSong D=420** | **Word2Vec D=300** |
|---|---|---|---|---|
| PQ4xfs (2D bits) | ~3% | ~3% | **>50%** | high |
| OPQ4xfs (2D bits) | ~2% | ~2% | **>100%** | **>200% (max)** |
| LSQ4xfs (2D bits) | unstable | unstable | unstable | unstable |
| **RaBitQ (D bits)** | **<2%** | **<2%** | **<40%** (max) | ~75% (max) |

→ MSong/Word2Vec 上 PQ-family 灾难失败（ANN recall ≤60% even with rerank）；RaBitQ 全 6 dataset 稳定 work。

### 工业组合方式（updated）

[per concepts/rabitq.md "工程实现要点"]

| 系统 | 当前 quantizer | RaBitQ 集成现状 |
|---|---|---|
| **Faiss** | PQ / OPQ / RQ / LSQ / PRQ / PLSQ / ScaNN | **未集成**（同年发表 2024）；社区 PR logical next step |
| **DiskANN** | PQ in DRAM + SSD full-precision rerank | **未集成**；理论上 RaBitQ 替换 DRAM PQ 后更准更紧但 graph 集成 challenge |
| **SPANN** | 不用量化（centroids + posting list 全精度 SSD） | **未集成**；理论上 posting list 用 RaBitQ 节省 SSD 4× 但 closure clustering 与 RaBitQ normalization 兼容性开放 |
| **Milvus** | IVF_PQ / IVF_SQ8 / SCANN / GPU CAGRA | **未集成**（v2.6.x 文档无 RaBitQ） |
| **Pinecone** | 黑盒（adaptive 自动选） | 未公开 |
| **VBASE** | IVFFlat 全精度（论文 §5.3） | **未集成**（论文 2023 早于 RaBitQ） |

→ RaBitQ 是 **research-stage**，无工业 production 集成——但**所有现有 vector DBMS 都是 logical adopter**。

### 与 RM iterator 的双层正交（NEW）

[per topics/topk-vs-iterator-model.md "K' 消除：双层路径"]

quantization 维度 + query interface 维度 **正交**：

```
                    K' 预测问题
                          │
            ┌─────────────┼─────────────┐
            ▼                            ▼
    Engine layer (VBASE)           Estimator layer (RaBitQ)
    Iterator + RM                  unbiased + sharp bound
    动态 K̃                          drop by lower bound
```

→ 理论上 VBASE engine + IVF + RaBitQ rerank 是 dual K' 攻击；wiki 内 zero coverage（VBASE 与 RaBitQ 论文同年发表，相互不知道）。

### Open / 未覆盖

- **RaBitQ + graph-based 索引（HNSW / Vamana / NSG）**：仍开放——RaBitQ §4 明示 future work
- **RaBitQ + GPU**：bitwise + popcount 在 GPU 是否仍快于 PQ LUT？未实测
- **RaBitQ + extreme high-D (D > 1000)**：现代 LLM embedding (OpenAI ada-002 1536-d) 未测；GIST 960-d 是论文上限
- **Sparse vector quantization**：所有 dataset dense；sparse 未测

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/scann.md](../../concepts/scann.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md](../../benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md)
