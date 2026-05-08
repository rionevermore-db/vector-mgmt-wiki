---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [topics/index-selection.md, topics/topk-vs-iterator-model.md, systems/vbase.md, concepts/relaxed-monotonicity.md]
cited-count: 4
---

# Post-snapshot (zhang-2023-vbase): scale-tier-shifts

## TL;DR (delta from patel-2024-acorn post)

**第 11 质变维度（NEW）：query interface（TopK vs Iterator）**——VBASE [zhang-2023-vbase] 暴露了一个**与 scale 部分耦合**的新质变：当 query workload 从单 column TopK 跨入 multi-column / range / Join 时，TopK 接口成为根本瓶颈（200-7900× 慢于 Iterator）。这与之前 10 个维度的"scale → 算法换"不同——是 **"workload 复杂度 → 接口范式换"**。前者由数据量驱动，后者由查询形态驱动。第 11 维度与前 10 维度**正交**（任何 scale 下都可触发，但在 multi-column/range/Join workload 显著）。

## Answer

### 十一个质变点（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化容忍度 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a (OLAP) / 8b (OLTP) 子分化 |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity 触发 |
| 10 | filter-aware build → predicate-agnostic build | filter cardinality | >1000 filters / 任意 operator |
| **11（NEW）** | **TopK interface → Iterator + RM** | **query 复杂度** | **multi-column / range / Join 出现** |

### 第 11 质变维度的特征（NEW）

[per topics/topk-vs-iterator-model.md "Q4-6 实测 + Q8 vector Join"]

| Query 形态 | 触发条件 | 推荐方案 |
|---|---|---|
| Single-vector TopK 无 filter | 推荐 / 简单相似搜索 | **TopK 接口足够**（VBASE Q1 与 PASE 算法等价） |
| Single-vector TopK + filter | static selectivity 已知 | TopK + partition-based (Milvus E) 或 4-plan CBO (ADBV) 均可 |
| **Multi-column TopK** | 多 vector field（image + text）综合排序 | **必须 Iterator + RM**（VBASE 比 Milvus iterative merging 200-300× 快） |
| **Vector Range Filter** | 阈值返回 / outlier 检测 | **必须 Iterator + RM**（K' 不可知） |
| **Vector Join** | similarity-based JOIN | **必须 Iterator + RM**（PG nested-loop 7900× 慢） |

→ workload 越复杂，TopK 接口成本越高；最简单的 Q1 上两接口算法等价。

### 第 11 维度与其他维度的正交性（NEW）

```
                              query interface (维度 11)
                              ↓
                        TopK             Iterator + RM
            scale 1B ─── Faiss/HNSW      VBASE-HNSW (理论)
            scale 100B ─ SPANN/Filtered  VBASE+SPANN (实证 §5.4)
            HCPS filter  ACORN (TopK)    VBASE+ACORN (理论可行未实证)
            千亿 + HCPS  完全空白         完全空白
```

→ 维度 11 在所有规模 / 所有 filter 复杂度下都可应用——**workload 驱动而非 scale 驱动**。

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| 质变维度数 | 10 | **11** |
| 维度的驱动因素 | 全部由 data 维度驱动（scale / cardinality / selectivity） | **+ workload 维度驱动（query 复杂度）** |
| 工程范式转换 | 算法替换（HNSW → Vamana → SPANN）| **+ 接口范式（TopK → Iterator）** |
| Wiki 内 production 实证 | 维度 1-10 的工业系统都已 ingest | **维度 11 仅 VBASE 学术原型，无 production 实证** |

### 第 11 质变维度的特殊性（NEW）

不同于维度 1-10：

- **维度 1-10 是"scale 推动"**：当数据更大 / filter 更多时**必须**换方案
- **维度 11 是"workload 推动"**：在任何 scale 下，**只要** query 形态复杂就触发

这意味着：
- 一个百万级 SaaS 应用如果做 vector Join / range filter，**第 11 维度先于 scale 维度触发**
- 一个十亿级 simple TopK 应用，**第 11 维度可能永不触发**（TopK 接口足够）

→ 维度 11 让 wiki 内"scale 推动"叙事被**workload 推动**叙事补充。

### 不算质变（参数微调）

- VBASE window size W=10 调
- VBASE sample rate 0.001 调
- Greedy vs Round-Robin 之 weight ratio 调

### 已知盲区

- **维度 11 的 production A/B 实证**：完全空白（VBASE 学术 only）
- **维度 11 + 维度 9/10 联合**：HCPS + Iterator + RM？理论上 ACORN 满足 RM，但 VBASE 未集成
- **维度 11 + scale 100B**：完全空白
- **维度 12+**：未来 ingest 是否暴露更多维度（如 multi-modal embedding 或 GPU 并行）

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/topk-vs-iterator-model.md](../../topics/topk-vs-iterator-model.md)
- [systems/vbase.md](../../systems/vbase.md)
- [concepts/relaxed-monotonicity.md](../../concepts/relaxed-monotonicity.md)
