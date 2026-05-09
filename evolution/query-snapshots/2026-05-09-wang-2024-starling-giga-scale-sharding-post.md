---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, systems/spann.md, systems/diskann.md, systems/milvus.md, queries/index-architecture-global-vs-routed.md]
cited-count: 5
---

# Post-snapshot (wang-2024-starling): giga-scale-sharding

## TL;DR (delta from gao-2024-rabitq post)

**Starling 把"千亿分片"决策从"单机 vs 多机"细化为"per-machine 单 segment vs per-machine 多 segment"**。BIGANN 1B 实证用 **31 segments × ~32GB RAM × 10GB disk**——这意味着 16 节点 × 1TB 私有云配置可能用 **~500 segments × 32GB** 而不是 16 × 1TB single-segment。**对查询 latency 的根本影响**：每 segment ANNS 5ms（Starling 实测）→ 跨 segment 协调 + 网络 → 仍可在 P99 < 50ms 内。**对 fault tolerance 的影响**：per-segment replicate 比 per-node replicate 更细粒度。但 Starling 单 query node 实证；分布式 query coordinator + 跨 node segment 协调未深入。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing 几千亿 + Faiss 1.5T | **+ Starling BIGANN 1B via 31 segments** |
| 分片粒度 | per-node | **per-segment（细粒度）** |
| Per-machine RAM 利用 | RaBitQ 推到 ~10B/node | **Starling 多 segment 共享 1TB RAM** |
| Fault tolerance | 跨 node replicate | **+ per-segment replicate（首次明示）** |

### Per-machine 多 segment 的工程意义（NEW）

[per systems/starling.md "Scale 边界" + §6.7, §6.11]

Starling §6.11 BIGANN 1B 实证：
- 31 segments × ~32 GB RAM each
- 10 GB disk per segment
- 2 query nodes total

→ **per-machine 多 segment** 是 Milvus 工程现实（[systems/milvus.md] 已 ingest）；Starling 把这变成 query-time 优势：

| 决策 | Single segment per node (1TB RAM full use) | Multi segment per node (1TB / N segments) |
|---|---|---|
| 每索引大小 | 大（~10B vec/node 装下）| 小（~33M vec/segment）|
| Build 并行 | 单实例顺序 | 多 segment 并行 |
| Update 影响 | 整索引 rebuild | per-segment rebuild |
| Fault tolerance | 整 node loss | per-segment loss |
| Query routing | 节点级 routing | segment 级 routing（query coordinator） |
| Index 优化潜力 | 单大索引 OR(G)/ℓ 难优化 | per-segment 极致优化（Starling 主场） |

→ **Per-machine 多 segment 是 vector DBMS 工程主流**（Milvus / Zilliz / Manu 都 segment-based）；Starling 是这个工程现实下的 disk graph 性能 unlock。

### 16 节点 × 1TB RAM 私有云的具体决策（updated）

[per queries/index-architecture-global-vs-routed.md + Starling §6.11]

**重新计算**：
- 千亿规模 768-d float32 = 300 TB raw vector
- 16 节点 × 1 TB RAM = 16 TB 内存预算（vs 300 TB raw → 仅能装 5%）
- **必须 disk-resident 路径** + per-segment 切分

| 决策 | 推荐 | Starling 影响 |
|---|---|---|
| 总体架构 | **(c) 层次路由 + per-segment partition** | **per-machine 多 segment 是细化** |
| 每 segment 容量 | ~33M vectors × 768-d uint8 ~25 GB raw | 同 Starling 实测 |
| 节点内 segment 数 | 1 TB / 32 GB per segment ≈ **30 segments/node** | 类似 Starling 1B BIGANN 配置 |
| 总 segment 数 | 16 × 30 = **480 segments** | 千亿 / 33M ≈ 3000 segments → 部分 segment 大于 33M |
| Disk 路径 | per-segment Starling | 关键差异：Starling 替代 DiskANN |
| Update 模式 | per-segment rebuild | 周级别全量重建在 Starling 下可行（build time ~1200s/segment × 480 / parallel） |
| P99 < 50ms | per-segment 5ms × ~6.3 segment dispatch ≈ 30 ms（理论） | OK |

→ Starling 的存在让 16 节点 × 1TB 配置可以走 segment-level（更细粒度）而非 single-server 大索引。

### Build time 估算（NEW）

Starling 单 segment 33M build ~1200s（BIGANN 实测）。

千亿 / 33M ≈ 3000 segments。**所有 segments 周级别全量重建**：
- 串行：3000 × 1200s = 100 小时（4 天）→ 超出周级别预算
- 16 nodes × 30 parallel segments = 480 并行 → 3000 / 480 × 1200s ≈ 7500s ≈ **2 小时**
- → **per-machine 30 segments 并行 build 可在 2 小时完成**——周级别更新轻松满足

### 与之前 ingest 的累积演进

| | patel-2024-acorn post | zhang-2023-vbase post | gao-2024-rabitq post | **wang-2024-starling post** |
|---|---|---|---|---|
| 分片决策 | (c) routed | 不变 | + per-node 容量 ↑ | **+ per-machine 多 segment 细化** |
| 单 segment 实证 | n/a | n/a | n/a | **33M × 4 datasets** |
| Billion-scale 实证 | SPANN/DiskANN single-server | n/a | n/a | **Starling 31 segments × 32GB BIGANN 1B** |
| Build time 估算 | (隐含) DiskANN 5+ 天 | n/a | n/a | **Starling per-segment 1200s × parallel = 周级别** |

### 已知盲区

- **千亿规模 × Starling 实证**：仅 BIGANN 1B（10 亿）；千亿（10^11）未实证
- **跨 node segment 协调**：单 node 多 segment 实证 OK；多 node + query coordinator + 网络 latency 未深入
- **Build time 在 SSD 容量约束下**：每 node 30 segments × 10 GB = 300 GB persistent disk，可行
- **Starling + filter / multi-vector heavy workload**：完全空白
- **HCPS + segment-level**：完全空白

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
