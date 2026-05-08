---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: gao-2024-rabitq
wiki-pages-total: 55
cited-pages: [topics/index-selection.md, topics/topk-vs-iterator-model.md, concepts/rabitq.md, concepts/product-quantization.md]
cited-count: 4
---

# Post-snapshot (gao-2024-rabitq): scale-tier-shifts

## TL;DR (delta from zhang-2023-vbase post)

**RaBitQ 不新增第 12 质变维度——而是细化第 11 维度（query interface）**。第 11 维度（TopK vs Iterator + RM）在 zhang-2023 ingest 时定义为 **engine 层** workload-driven shift；RaBitQ 提供 **distance estimator 层** 的 dual 攻击点——同样针对 K' 预测问题但作用层不同。所以第 11 维度从单层升级为**双层 query-engine + distance-estimator 协同**。**对 scale-tier 决策的影响**：在 1B+ scale + memory budget 严的场景，RaBitQ 替代 PQ 后 (a) DRAM 占用降到 1/2（D bits vs 2D bits），(b) 估计精度反而更好——这让"维度 2/3 单机 RAM/SSD 触顶"的临界点**向后推 ~2×**（同 RAM 下能装 2× 数据）。

## Answer

### 十一个质变点（updated with RaBitQ sub-dimension）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B（**RaBitQ 推到 ~2B**） |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化容忍度 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a (OLAP) / 8b (OLTP) 子分化 |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity 触发 |
| 10 | filter-aware build → predicate-agnostic build | filter cardinality | >1000 filters / 任意 operator |
| **11** | **TopK interface → Iterator + RM** (engine layer) | **query 复杂度** | **multi-column / range / Join 出现** |
| **11b（NEW sub）** | **Biased PQ → Unbiased + sharp error bound (estimator layer)** | **quantizer 自身的理论保证** | **同样攻击 K' 问题，但 quantizer 层** |

### 第 11b sub-dimension 的特征（NEW）

[per concepts/rabitq.md + topics/topk-vs-iterator-model.md "K' 消除：双层路径"]

```
        K' 预测问题（第 11 维度的本质）
                  │
        ┌─────────┼─────────┐
        ▼                    ▼
   Engine layer         Estimator layer
   (VBASE 2023 OSDI)    (RaBitQ 2024 SIGMOD)
        │                    │
   Iterator + RM         Unbiased + bound
   动态 K̃                drop by bound
```

→ **第 11 维度不是单点而是双层**——之前 zhang-2023 ingest 时只看到 engine 层；RaBitQ 暴露 estimator 层是 dual。两条路径**正交可叠加**：理论上 VBASE engine + IVF + RaBitQ rerank 是完整 K' 消除栈（wiki 内 zero coverage 的 frontier）。

### 对维度 2 (单机 RAM 触顶) 的影响（NEW）

[per concepts/rabitq.md "Code length"]

| Quantizer | Code 长度 | 1B SIFT 1024-d 内存 |
|---|---|---|
| Flat (no quantization) | 32D bits = 32×1024 bits | 4 TB |
| PQ default | 2D bits = 2048 bits | 256 GB |
| **RaBitQ default** | **D bits = 1024 bits** | **128 GB** |

→ RaBitQ 单机 1.5 TB RAM 服务器（如 [Milvus](../../systems/milvus.md) 1.x 实测的 ecs.re6.26xlarge）能装 **~10B vectors**（vs PQ ~5B）——维度 2"单机 RAM 触顶"临界点向后推。

但这是**容量**改进而非范式转换——所以仍是维度 2 内的细化，不是新维度。

### 对维度 3 (SSD 触顶) 的影响

理论上同样：[SPANN](../../systems/spann.md) posting list 用 RaBitQ 替代全精度后 SSD 占用 1/4（32D → D bits）。但 SPANN 论文 §3 明确反对量化（"recall 上限不被量化限制"）；RaBitQ 的 sharp error bound 让 SSD 全精度 re-rank 仍可用——这是 future work。详见 systems/spann.md Open Q。

### 与之前 ingest 的演进

| | zhang-2023-vbase post | **gao-2024-rabitq post (NEW)** |
|---|---|---|
| 质变维度数 | 11 | **11 (with 11b sub-dimension)** |
| 维度 11 的层级 | engine layer 单层 | **engine + estimator 双层** |
| K' 消除路径 | 单一（VBASE iterator） | **双层正交（VBASE engine + RaBitQ estimator）** |
| 维度 2 RAM 临界点 | 1B（PQ 假设） | **2B+（RaBitQ 推后）** |

### 不算质变（参数微调）

- RaBitQ ε₀ = 1.9 调（论文实测 1.9 在 6 dataset 都饱和——零调参）
- RaBitQ B_q = 4 调（饱和点）
- code length D vs 2D padding（RaBitQ 推荐 D；调 2D 额外精度 marginal）

### 已知盲区

- **维度 11 双层叠加 (VBASE + RaBitQ) 实证**：完全空白——两个论文相互不知道，未实证
- **维度 11b + scale 100B**：完全空白
- **维度 11b + filter heavy (HCPS)**：完全空白
- **维度 12+**：未来 ingest 是否暴露更多维度（如 multi-modal embedding 或 GPU 并行）

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/topk-vs-iterator-model.md](../../topics/topk-vs-iterator-model.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
