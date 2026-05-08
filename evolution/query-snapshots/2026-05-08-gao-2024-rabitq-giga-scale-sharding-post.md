---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [concepts/rabitq.md, systems/spann.md, systems/diskann.md, topics/disk-vs-memory-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (gao-2024-rabitq): giga-scale-sharding

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 不直接给出新的千亿/万亿分片策略**——research prototype 仅测到 ≤2.34M。但 RaBitQ 的 D-bit code（PQ 默认的一半）+ unbiased estimator 让"per-node 容量"理论上**翻倍**——16 节点 × 1TB RAM 私有云配置下，RaBitQ + IVF 在每节点装 ~10B vectors 而 PQ 仅 ~5B。这把"维度 2 单机 RAM 触顶"的临界点**向后推 ~2×**。但 RaBitQ + 分布式 / SSD / billion-scale 全部 zero coverage——所以工程实践仍走 SPANN / DiskANN / Faiss 路径。**Q1 的具体决策不变**：层次路由 + SPANN-style partition 是 production 实证唯一答案。

## Answer

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing 几千亿 + Faiss 1.5T | **不变** |
| Per-node 容量 | PQ 假设 ~5B/节点（256 GB RAM） | **+ RaBitQ 假设 ~10B/节点（同 RAM）** |
| Quantization 选择 | PQ default | **+ RaBitQ as alternative** |
| 分布式集成 | 同前 | **不变**（RaBitQ 单实例） |

### Per-node 容量改进（NEW）

[per concepts/rabitq.md "Code length"]

768-d vector：

| Quantizer | Code 长度 | 1B vectors 内存 | 1 TB RAM 容量 |
|---|---|---|---|
| Flat 32-bit float | 32D bits = 24 KB/vec | 24 TB | ~40M vectors |
| PQ default 2D bits | 2D bits = 1.5 KB/vec | 1.5 TB | ~600M vectors |
| **RaBitQ default D bits** | **D bits = 96 bytes/vec** | **96 GB** | **~10B vectors** |

→ 16 节点 × 1 TB RAM = 16 TB 内存预算下：
- PQ：~10B vectors （远小于"千亿"）
- **RaBitQ：~160B vectors （能装千亿）**

但有 caveat：
- 实际还需 (a) IVF centroids 占用，(b) graph adjacency 占用，(c) 操作系统 / 索引 metadata
- 实际容量 ~30-50% 上面理论值

→ **RaBitQ 把千亿分片"压力"从分布式转移回单节点 + quantization**——但需要 RaBitQ 在 billion-scale 实证（gap）。

### 16 节点 × 1TB RAM 私有云的具体决策（updated）

[per queries/index-architecture-global-vs-routed.md + 不变的 production 实证]

| 决策 | 推荐 | RaBitQ 影响 |
|---|---|---|
| 总体架构 | **(c) 层次路由（SPANN-style 或 Milvus segment）** | 不变 |
| 索引类型 | **IVF + 量化**（千亿规模 graph 全内存装不下） | RaBitQ 替代 PQ |
| 量化器 | PQ / OPQ default | **RaBitQ if billion-scale 实证后** |
| 存储介质 | **DRAM + SSD 混合**（SPANN-style centroids in DRAM + posting on SSD） | **不变**（RaBitQ 在 SSD 路径上 zero coverage） |
| 分片策略 | **balanced clustering + bin-packing**（SPANN §4.3） | 不变 |
| Update 频率 | **周级别 full rebuild** 接受 | RaBitQ 重 sample P 矩阵 + 重 quantize |
| P99 < 50ms | SPANN 90% recall ~1ms 单机 + cross-shard ~6.3 节点 dispatch | RaBitQ 估计速度 3× faster than PQ → 可能进一步降 |

→ **总体架构不变**：千亿规模 SPANN-style partition + DRAM/SSD 混合仍是工业实证唯一答案。RaBitQ 是 quantization layer 的**潜在升级**——但需 billion-scale 实证 gap 关闭。

### 已知盲区

- **RaBitQ + 1B+ scale 实证**：完全空白（论文 max 2.34M）
- **RaBitQ + 分布式**：完全空白
- **RaBitQ + SSD posting list (SPANN)**：完全空白
- **RaBitQ + DiskANN PQ 替换**：完全空白
- **RaBitQ + 周级别 rebuild 流程**：理论可行（RaBitQ index time 与 PQ 同量级——GIST 117s vs PQ 105s）但 production validation 缺
- **HCPS + 千亿 + RaBitQ**：完全空白

## Cited Pages

- [concepts/rabitq.md](../../concepts/rabitq.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
