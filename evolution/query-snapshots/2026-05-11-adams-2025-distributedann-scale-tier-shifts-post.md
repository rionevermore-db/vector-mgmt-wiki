---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/spann.md, systems/spfresh.md, systems/diskann.md, systems/turbopuffer.md, concepts/vamana.md, topics/disk-vs-memory-ann.md]
cited-count: 7
---

# Post-snapshot (adams-2025-distributedann): scale-tier-shifts

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN 给 wiki Tier 5 (≥1T) 提供首个 published 详细 architecture**——Microsoft Bing "hundreds of billions of vectors" 之前是 implied (SPANN paper §1 末尾), 现在 DistributedANN paper §4-5 给出 detailed SKU / latency / IO numbers + 与 SPANN-style head-to-head Table 1. **关键 NEW**: **Tier-shift 新维度**——同 scale 内**"distributed single graph vs distributed partitioned graphs"是新的二选一**。Bing 选择 single graph 的 trade-off：+7.8pp recall, 6× throughput, 7.5× less IO 换 2.9× more SSD, 1.6× p50 latency. **Tier 6 future (>10T) hint**：论文 §5.1 提"dense cluster + fully connected network → shared memory pool"——下一代 architectural extension 已 outlined.

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| Tier 5 (≥1T) production data points | Turbopuffer 3.5T+ (commercial SaaS, OSS-known) + Bing 1.5T (SPANN implied) | **+ Bing hundreds of billions (DistributedANN paper-documented detailed)** |
| Tier-shift driver dimensions | scale → memory/disk + index choice + parallel | **+ same-scale "single graph vs partitioned graphs" 二选一 (新维度)** |
| Single graph 上限 | 之前 ~10B (HNSW) | **50B single graph (DistributedANN @ Bing)** |
| Future tier (>10T) | wiki 未涵盖 | **§5.1 outlines next steps (dense cluster + fully connected + shared memory)** |

### 标准 tier-shift 表（updated 2026-05-11 post adams-2025）

| 规模 | 索引选择质变点 | 存储介质质变点 | 并行能力质变点 |
|---|---|---|---|
| ≤ 10 亿 (≤ 1B) | HNSW / IVF + memory all-in | 全内存 | 单机或 2-3 节点 |
| 10 亿 - 百亿 (1B-10B) | HNSW + quantization | 内存边界 临近 SSD | 多节点分 shard |
| 百亿 - 千亿 (10B-100B) | SPANN / SPFresh / DiskANN tier shift | NVMe / object storage | 路由层成必需 |
| **千亿 - 万亿 (100B-1T)** | **+ DistributedANN single graph option** (Bing 50B per slice) | NVMe + 高 IOPS / S3 (Turbopuffer) / **KV store as shared disk (DistributedANN)** | **single graph 跨 1000+ machines + sublinear scaling vs P × log(|X|/P) partition** |
| ≥ 1T | Bing **hundreds of billions** (DistributedANN paper-detailed) + Turbopuffer 3.5T+ (OSS-known) | KV store sharded + NVMe per host | **数千 machines coordinated via orchestration + KV store sharding** |
| Future >10T | §5.1 outlined: dense cluster + fully connected network + shared memory pool | Computational storage device at KV host | Kernel-bypass + GPU head index |

### Tier 5 内"single graph vs partitioned" 新二选一（NEW）

[per adams-2025-distributedann Table 1 + §4.4]

Tier 5 (≥100B) 内 production 选择不再唯一"分 partition"——出现**single graph distributed via KV store** 新选项. Same hardware footprint head-to-head:

| Tier-5 选择 | Recall@5 | Latency p50 | Throughput | SSD | 适用 |
|---|---|---|---|---|---|
| Clustered partitioning (SPANN-style) | 83.0% | **16 ms** | ~15K QPS | **270 TiB** | Latency-critical workload |
| **DistributedANN single-graph distributed** | **90.8%** | 26 ms | **>100K QPS** | 780 TiB | Throughput + recall priority |

**Bing 选 DistributedANN**: 6× throughput + 7.8pp recall + 7.5× less IO 是优于 1.6× p50 latency penalty 的核心驱动. 这是 hardware footprint same 但每 host 用更多 SSD/memory 的 trade-off.

### MSR 谱系 5 代演化贯穿 tier-shift（NEW）

[per adams-2025-distributedann + 谱系 5 论文]

MSR 5 代演化映射到 tier-shift:
- **Tier 1 (≤1B)**: DiskANN 2019 single-node 是默认
- **Tier 2-3 (1-100B)**: SPANN 2021 引入 partition + centroid, Bing 21-24 主力
- **Tier-spin: streaming**: FreshDiskANN 2021 / SPFresh 2023 给 streaming + in-place 更新
- **Tier 4-5 (100B-1T)**: DistributedANN 2025 single-graph distributed, Bing 25 主力
- **Tier 6 (>1T) future**: §5.1 outline

每代演化 focus 不同 axis (single→partitioned→streaming-graph→streaming-cluster→distributed-graph)——Bing 当前 production 在 graph 路径的 distributed 形态.

### Tier-shift 决策驱动（updated 2026-05-11 post adams-2025）

1. **Tier 1 (≤1B)**: HNSW + memory 默认
2. **Tier 2 (1-10B)**: 加 quantization
3. **Tier 3 (10-100B)**: 切 SPANN / DiskANN / Starling / SPFresh
4. **Tier 4 (100B-1T)**: **同 scale 内二选一**: clustered partitioning (latency 优先) **OR DistributedANN (throughput + recall 优先)**
5. **Tier 5 (≥1T)**: 多 slice + cross-slice orchestration; Bing DistributedANN + Turbopuffer SPFresh 是两个独立 production 数据点
6. **Future (>10T)**: dense fully-connected cluster + shared memory pool + kernel-bypass + GPU head index

### 已知盲区

- **DistributedANN cross-slice query**: Bing hundreds of billions = multiple slices; query 跨 slice 协调 paper 不深入
- **>10T 实际 production**: 仍无 vendor 公开 ≥10T 数据点; Bing total 是 "hundreds of billions" not >1T
- **DistributedANN single-machine viable threshold**: DiskANN single-node 1B 是 hard limit; DistributedANN 多 host 上限随 KV store scale
- **Tier-shift cost analysis**: 不同 tier vendor SKU 成本 / W / IOPS 比较 wiki 未量化

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
