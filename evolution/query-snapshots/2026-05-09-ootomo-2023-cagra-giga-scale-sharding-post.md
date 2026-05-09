---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, concepts/cagra-graph.md, topics/gpu-vs-cpu-ann.md, queries/index-architecture-global-vs-routed.md]
cited-count: 4
---

# Post-snapshot (ootomo-2023-cagra): giga-scale-sharding

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 不直接适用千亿 read-heavy + 内存预算严的 16 节点 × 1TB RAM 私有云配置**——CAGRA single-A100 仅支持 ~100M vectors (96-d) 或 ~24M (768-d)，且 768-d 千亿（1000 segments × 100M）远超合理 GPU 集群规模。**对该 query 配置的影响**：私有云部署多数仍 CPU + SSD 路径；GPU 仅在**特定 hot segment**（high-traffic, low-latency 要求）上 deploy CAGRA 作"加速层"。**架构形态**：cluster routing → per-segment SPFresh/Starling on CPU+SSD → 选择 hot 5-10% segments deploy CAGRA on GPU。但 GPU + CPU 混合 cluster 的 routing 协议是 wiki 内空白。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing / Starling 31 segments / FreshDiskANN 1B ramp-up | **不变（CAGRA 不直接适用）** |
| Per-machine GPU 选项 | 完全空白 | **+ NVIDIA A100/H100 GPU as hot segment accelerator** |
| 私有云 GPU 部署 | 完全空白 | **+ 推论：hot segment GPU CAGRA + cold segment CPU + SSD hybrid** |

### CAGRA 在 16 节点 × 1TB 私有云配置的可行性（NEW）

[per systems/cagra.md "Scale 边界"]

千亿 768-d float32 = 300 TB raw。每 GPU A100 80GB 容量 ~24M vectors (768-d float32) 或 ~48M (FP16)。

**理论上**：千亿 / 24M = ~4000 GPU——unrealistic。

**实际**：CAGRA 不竞争千亿全量部署；适合**hot subset**（频繁查询的 5-10% 数据 + low latency 要求）：

```
私有云 (16 nodes × 1TB RAM) Hybrid 架构（推论）:

  Hot segments (5-10% data, 高 query traffic):
    GPU servers (1-2 A100 each) running CAGRA
    Per A100: 24M-48M vectors (768-d FP16)
    总 hot capacity: 16 × 8 × 48M = ~6 billion vectors

  Cold segments (90-95% data):
    CPU + SSD (DiskANN / SPFresh / Starling per segment)
    每 node 处理 ~6 billion vectors (1000 segments × 6M each)
    总 cold capacity: 16 × ~6 billion = ~100 billion (千亿)
```

→ **混合 GPU + CPU 部署是千亿合理路径**——但 routing 协议（query coordinator 决定 hot vs cold path）是 wiki 内 zero coverage。

### 千亿配置中 CAGRA 的角色（NEW）

| Workload 模式 | 推荐路径 |
|---|---|
| 全量 cold scan (analytics, batch) | CPU + SSD (DiskANN / SPANN) |
| Top 5% hot dataset 高频 query | **GPU CAGRA + per-GPU 24-48M vectors** |
| Streaming insert into hot subset | streaming 路径不直接（CAGRA static）→ **临时 buffer 在 CPU FreshDiskANN, 周期 promote to GPU CAGRA** |
| Long-tail cold query | CPU SPFresh / Starling |

→ GPU CAGRA 在千亿场景下是"hot path accelerator"而非"primary index"。

### 与之前 ingest 的累积演进

| | wang-2024-starling post | gao-2024-rabitq post | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|---|---|
| 千亿 hardware mix | per-node 多 segment CPU+SSD | + RaBitQ memory efficiency | + streaming dimension | **+ GPU hot accelerator option** |
| Per-machine 容量 | 30 segments × 32GB (Starling) | RaBitQ ~10B/node | FreshDiskANN 8 × 128GB | **+ GPU node ~6B/A100 (768-d FP16)** |
| Update model | static rebuild | quantizer fast rebuild | streaming insert/delete | **CAGRA static → 周期重 build** |
| Latency budget P99<50ms | ~30 ms total | + RaBitQ 速度提升 | + streaming overhead | **+ GPU hot path 极快 (<5 ms)** |

### 千亿 + read-heavy + low latency 决策（updated）

| 决策 | 推荐 | CAGRA 影响 |
|---|---|---|
| Cluster topology | (c) 层次路由 | 不变 |
| Per-machine partition | per-node 多 segment | + **GPU subset for hot segments** |
| Per-segment disk index | Starling / DiskANN / FreshDiskANN | + **GPU-accelerated for hot 5-10%** |
| Update model | 周级 rebuild + streaming | + **CAGRA static rebuild on GPU when promoted to hot tier** |
| Latency optimization | NVMe SSD + cache | + **GPU CAGRA <5 ms hot path** |
| Cost optimization | CPU + SSD cluster | + **GPU 部署仅 hot tier (5-10% nodes)** |

### 已知盲区

- **GPU + CPU hybrid cluster routing 协议**：完全空白
- **Hot/cold segment dispatch logic**：未深入
- **GPU streaming graph (FreshCAGRA) 千亿 hot tier**：理论可行但 CAGRA 不支持 streaming
- **Multi-GPU CAGRA sharding 大规模实证**：论文 §IV-C 仅提及
- **NVIDIA H100 / Blackwell 千亿场景**：H100 HBM3 + bandwidth 远高 A100，未量化
- **GPU 路径与 [Pinecone serverless](../../systems/pinecone.md) 商用 GPU 集群对比**：完全空白
- **AWS Inferentia / Google TPU 上 CAGRA-equivalent**：完全空白

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
