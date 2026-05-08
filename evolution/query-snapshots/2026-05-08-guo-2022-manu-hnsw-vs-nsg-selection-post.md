---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/milvus.md, benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md]
cited-count: 4
---

# Post-snapshot (guo-2022-manu): hnsw-vs-nsg-selection

## TL;DR (delta from pinecone-docs post)

**Manu 论文实测 HNSW 与 IVF-FLAT 双双系统击败四个开源 vector engine**（Elasticsearch / Vearch / Vald / Vespa），但**Manu 论文 §5 没把 NSG 放入对手**——Manu 内置 NSG 索引但实验对手用 Vald (NGT) / Vespa (HNSW)。所以 HNSW 在 Manu 中是**实测最强 graph 选择**，NSG 在论文中无新数据。**RNSG 身份歧义部分缓解**：Manu 论文表中明确列 NSG 与 HNSW 并列（不是"RNSG"），意味着 SIGMOD 1.x 的"RNSG"歧义可能是 SIGMOD 论文写作时的简称，Manu (2.x) 中正式名为 NSG。

## Answer

### Manu 论文的索引族（NEW）

[per systems/milvus.md "v2.x 索引家族" + guo-2022-manu Table 1]

Manu Table 1 明确列：

| 类别 | 索引 |
|---|---|
| Vector Quantization | PQ / OPQ / RQ / SQ |
| Inverted Index | IVF-Flat / IVF-PQ / IVF-SQ / IVF-HNSW / IMI |
| **Proximity Graph** | **HNSW / NSG / NGT** |
| Numerical Attribute | B-Tree / Sorted List |

→ NSG 在 Manu 中**不是 RNSG**——直接命名 NSG。但实验只用 HNSW + IVF-FLAT。

> **wiki 解读**：解决 wang-2021-milvus 论文的 RNSG 命名歧义（"RNSG"是 NSG 还是 Rand-NSG）。Manu (2.x) 论文表中 NSG 是 Fu 2017 NSG，与 Vamana / Rand-NSG 不同——分别命名。

### Manu §5 实测中 HNSW 数据（NEW）

[per benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md "主结果"]

SIFT10M / DEEP10M：
- **Manu HNSW 在 throughput-recall 曲线上排第一**（>8000 QPS @ 0.95 recall）
- Manu IVF-FLAT 排第二
- Vald NGT、Vespa HNSW、Vearch HNSW 都被压制
- 论文论证：Manu 用 cache-aware + SIMD 优化让同算法实现更优

→ Manu 实证：**给定 SIFT10M / DEEP10M scale，HNSW 是最优 graph 选择**。NSG 未参评但 Milvus 仍内置（使用率较低）。

### 选型决策树更新（与之前 post 一致）

| 场景 | 推荐 | 备注 |
|---|---|---|
| 静态 + million-scale + 高 precision | NSG（NSG 论文实测最强）或 Manu HNSW（实证 throughput 最强） | 看具体 workload |
| 增量 add | HNSW | NSG 不增量 |
| Million-100M Manu/Milvus 部署 | HNSW（默认 graph） | Manu 实测最优 |
| Billion-scale + SSD | DiskANN 或 [Manu SSD index](../../concepts/manu-ssd-hierarchical-kmeans.md) | NeurIPS 2021 winner |
| 持续 update + cluster-based | SPFresh LIRE | in-place |

### 与之前 ingest 的演进

| | wang-2021 / milvus-docs / xu-2023-spfresh post | **guo-2022-manu post (NEW)** |
|---|---|---|
| RNSG 身份 | wang-2021 提出歧义；milvus-docs DISKANN 单列暗示 RNSG ≠ Rand-NSG | **Manu 论文 Table 1 直接命名 NSG，与 NGT / HNSW 并列**——歧义关闭 |
| HNSW vs IVF 实测 | Milvus 1.x 论文给 SIFT10M HNSW 数据 | **Manu 论文 + 4 baseline 对比** |
| Cache-aware / SIMD 实证 | Milvus 1.x 论文给 | Manu 重新实证（更大胜幅） |

### Open / 未覆盖

- **NSG vs HNSW 在 Manu 中直接对比**：Manu §5 未做（仅用 HNSW 与 IVF-FLAT 实验）
- **Manu NGT 集成**：Table 1 列出但 §5 未实测；NGT 是 [Yahoo Japan 的 graph 索引]
- **graph-based + in-place update**：仍未解（Manu 不解决 graph in-place）

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md](../../benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md)
