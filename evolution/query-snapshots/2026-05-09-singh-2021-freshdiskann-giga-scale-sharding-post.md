---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, systems/spfresh.md, systems/diskann.md, topics/in-place-vs-out-of-place-updates.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (singh-2021-freshdiskann): giga-scale-sharding

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 重新定义"周级别全量更新"的工程含义**——之前 wiki 假设"周级别 full rebuild"是 baseline，FreshDiskANN 的 StreamingMerge 让"持续增量"替代周级 batch 成为**首选**。**对 16 节点 × 1TB RAM 私有云的影响**：(a) 周级别全量更新可改为持续 streaming + 周级 StreamingMerge consolidation（5.25× 比 full rebuild 快），(b) 每节点 1 TB RAM 可装 8 segments × 128 GB FreshDiskANN per node（vs Starling 30 segments × 32 GB）——graph path 选择是 segment 数量减少但每 segment recall 上限更高。**跨节点 fault tolerance** + segmented routing 仍未覆盖，是 wiki frontier。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing 几千亿 + Starling 31 segments × 32GB BIGANN 1B | **+ FreshDiskANN ~128GB per index, single-machine 800M streaming demo** |
| Update 模型 | 周级全 rebuild（DiskANN + Starling 模式） | **+ Streaming merge（5.25× faster than rebuild）** |
| Per-node segment 数 | Starling 30 × 32 GB | **+ FreshDiskANN 8 × 128 GB（更大 segment + 更高 recall ceiling）** |
| Fault tolerance | per-node replicate (DiskANN) / per-segment (Milvus) | 不变 |

### "周级别全量更新"重新解读（NEW）

[per topics/in-place-vs-out-of-place-updates.md + benchmarks/freshdiskann-streaming-sift800m.md]

之前 wiki 把"周级别全量更新"当成系统约束。FreshDiskANN 揭示这是**实现选择**：

| 实现路径 | 成本 |
|---|---|
| **Periodic full rebuild** (DiskANN baseline) | 1100 GB DRAM + 32 cores × 2 天，整 segment block 服务 |
| **FreshDiskANN streaming + 周级 StreamingMerge** | 持续 ~128 GB + low CPU；merge 4.4h（800M 每 30M change）；不阻塞 search |
| **SPFresh + LIRE in-place rebalance** | 持续 ~10 GB + 2 cores；不需要 merge cycle |

→ "周级别更新"在 FreshDiskANN 下变成"周级别 merge consolidation"（不是停服全 rebuild）；在 SPFresh 下变成"连续后台 in-place"（甚至不需 merge cycle）。

### 16 节点 × 1TB RAM 私有云：双路径选择（NEW）

[per queries/index-architecture-global-vs-routed.md + Starling/FreshDiskANN per-segment cost]

| 决策因素 | Cluster path（SPFresh）| Graph path（FreshDiskANN）|
|---|---|---|
| 数据装载 | per-node ~250B vec via 4GB SPFresh × N segments | per-node ~80B vec via 128GB FreshDiskANN × N segments |
| 总集群容量（16 nodes） | ~4 trillion vec（远超千亿） | ~1.3 trillion vec |
| Recall ceiling | ~95% (cluster centroids constraint) | ~98%+ (graph search) |
| Streaming insert latency | <1ms（LIRE 触发率 0.4%） | ~1ms（FreshVamana） |
| Streaming delete latency | <1μs（lazy delete） | <0.1μs（lazy delete） |
| Search latency | ~5-10ms steady（LIRE NPA stable） | ~5-15ms steady（FreshVamana + LTI 双源 search） |
| Search latency during merge | n/a（无 merge cycle） | spike 25-40ms during Patch phase |
| Fault tolerance unit | per-segment | per-segment |
| Build time per segment | n/a（in-place 无 build phase） | per-segment ~1200s for 33M（Starling 数据，FreshDiskANN 类似） |
| 千亿规模可行性 | **更经济** | **更高 recall** |

→ **千亿 + 高 recall** → FreshDiskANN per-segment（每 segment 800M-1B）；**千亿 + cost-sensitive** → SPFresh per-segment（更小内存预算）。

### Build time 估算（updated）

千亿 / 1B per segment ≈ 100 segments。**所有 segments 周级别 streaming + consolidation**：

| Path | Cost per segment per week | 总 cost |
|---|---|---|
| Static rebuild (DiskANN) | 23 h × 100 = 2300 h | unrealistic |
| **FreshDiskANN StreamingMerge** | **4.4 h merge per segment + continuous insert/delete** | **realistic** within 16 nodes (parallel ~6 segments per node × 4.4 h ≈ 26 h per cycle) |
| SPFresh LIRE | continuous in-place, no merge cycle | trivial |

→ FreshDiskANN 把"周级别 update"从"5+ 天 unrealistic" 变成"~26 h realistic"——这是 graph-path 在 streaming 千亿场景的关键解锁。

### 与之前 ingest 的累积演进

| | gao-2024-rabitq post | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|---|
| Per-machine 容量 | + RaBitQ ~10B/node | + Starling 30 segments × 32GB | **+ FreshDiskANN 8 × 128GB streaming-ready** |
| Update 模型 | 同前 | + 周期 block shuffling | **+ Streaming merge 5.25× faster** |
| 千亿可行性 | 需 (c) routing | per-segment 实证 | **+ streaming merge 让 weekly update 真正 realistic** |
| Fault tolerance 单元 | per-server | per-segment | **不变** |

### 已知盲区

- **千亿 × FreshDiskANN 实证**：仅 800M / 1B ramp-up；千亿（10^11）未实证
- **跨 node FreshDiskANN coordinator**：单 node FreshDiskANN demo OK；跨 node update routing + search aggregation 未深入
- **FreshDiskANN per-segment 在 Milvus segment 模型下的集成**：理论 fit Milvus 2GB+10GB segment——但 FreshDiskANN 默认 ~128 GB RAM 是 single-machine 大 index 假设，与 Milvus segment 模型不直接 match
- **HCPS + 千亿 + streaming**：完全空白
- **跨 path 组合 (graph + cluster) 在不同 segment**：理论上一台 node 内可以一些 segment 用 SPFresh + 一些 segment 用 FreshDiskANN（按 workload 选择）；wiki 内 zero coverage

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
