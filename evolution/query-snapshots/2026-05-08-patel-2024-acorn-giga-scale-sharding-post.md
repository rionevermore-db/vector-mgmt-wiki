---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [concepts/acorn.md, concepts/filtered-vamana.md, benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md, topics/attribute-filtering.md, systems/milvus.md]
cited-count: 5
---

# Post-snapshot (patel-2024-acorn): giga-scale-sharding

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**ACORN 实测最大 25M LAION**（单机 m5d.24xlarge）——比 FilteredVamana 28M DANN 略小但同量级。对千亿规模，ACORN **未直接实证**——但 predicate-agnostic + HCPS 支持的特性意味着 **filter set 极大的真实 web search workload**（如 Microsoft 广告 47 region + 千万 keyword 组合）下，ACORN 是唯一可考虑的方案。给定 16 × 1TB 千亿场景：若 workload 含 unbounded ad-hoc filter（regex / contains / between / OR），**ACORN 是 wiki 内唯一可行的 graph-based 方案**——FilteredVamana 在 HCPS 下完全 fail。

## Answer

### ACORN 实测 scale 的局限（NEW）

[per benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md "构造开销"]

| Dataset | Scale | ACORN-γ TTI | ACORN-γ Index Size |
|---|---|---|---|
| TripClick | 1.06M | 9902.9 s | 4.9 GB |
| LAION 1M | 1M | 835.8 s | 2.4 GB |
| **LAION 25M** | **24.65M** | **38007.5 s（10.5 小时）** | **59 GB** |
| SIFT1M | 1M | 148.9 s | 0.98 GB |
| Paper | 2M | 255.6 s | 2.5 GB |

→ ACORN-γ 在 LAION 25M 单机 m5d.24xlarge build 10+ 小时——仍可承受。但 1B 推断 TTI ~400 小时（17 天）——单机不可行。

### 16 节点千亿场景的 ACORN 推断（NEW）

理论可行性：
- 16 节点每节点 ~6.25B vectors
- ACORN-γ TTI 6.25B × 估计 ~10000 s/M ~10^11 s（不可行）
- ACORN-1 build TTI ~6.25B × ~250 s/M ~1.5×10^9 s ~480 days——仍不可行

**实际只能用 ACORN-1 + 应用层分片**：
- 每节点 ~390M（远小于 25M LAION 实测）
- 16 × ACORN-1 实例 + 应用层 routing
- 但 cross-node ACORN 路由 zero coverage
- 周级 update：ACORN-1 incremental 友好（HNSW-style add 加 2-hop 扩展）

→ **ACORN 千亿规模实证仍未填补**——但是 wiki 已 ingest 系统中 **HCPS workload 唯一可考虑**的选项。

### 与 FilteredVamana 的 production-readiness 对比（NEW）

[per benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md, filtered-diskann-vs-milvus-faiss-nhq.md]

| | FilteredVamana | ACORN |
|---|---|---|
| Scale 实证 | 28M DANN | 25M LAION |
| Production deployment | **Microsoft 广告 A/B +35-49% revenue** | 学术 only（Stanford research） |
| Filter cardinality | ≤1000 | **>10^11** |
| Operator | equality only | regex / contains / between / OR |
| 构造时间 | ~100s for 1M | **9900s for 1M (TripClick)** |

→ **production deployment**: FilteredVamana 显著领先（Microsoft 实测 P-value 0.009-0.03）；**filter capability**: ACORN 显著领先（HCPS 全覆盖）。

→ **千亿 production**: 若 workload 类似 Microsoft 47 region filter（LCPS scale），FilteredVamana 实证可达；若是 Pinecone-like ad-hoc keyword/regex 场景（HCPS），仅 ACORN 路径可行**（但 production scale 仍 zero 实证）**。

### 与之前 ingest 的演进

| | gollapudi-2023-filtered-diskann post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| Filter-aware 选项 | FilteredVamana / StitchedVamana 限 ≤1000 equality | **+ ACORN unbounded + 任意 operator** |
| HCPS 实证 | 不支持（FilteredVamana fail） | **+ ACORN 25M LAION** |
| Production 实证 | Microsoft +35-49% revenue | **学术 only（无 A/B test）** |
| Build 时间 | FilteredVamana ~100 s/M | **ACORN-γ ~10000 s/M（100×慢）** |

### 已知盲区

- **ACORN 千亿规模实证**：仅 25M
- **ACORN production A/B test**：仅 academic benchmark
- **ACORN 多节点分布式**：单机算法
- **2026 SIGMOD 跨 model 整合**：talk 当日 demo

## Cited Pages

- [concepts/acorn.md](../../concepts/acorn.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [systems/milvus.md](../../systems/milvus.md)
