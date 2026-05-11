---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/vamana.md, systems/distributedann.md, systems/diskann.md, systems/spann.md]
cited-count: 6
---

# Post-snapshot (adams-2025-distributedann): hnsw-vs-nsg-selection

## TL;DR (delta from turbopuffer-docs post)

**Vamana (DiskANN-family) graph 通过 [DistributedANN](../../systems/distributedann.md) 进入 wiki 内**最大 production scale graph-based ANN 实证**——50B × 384-d int8 single graph across 1000+ machines at Microsoft Bing。这把 Vamana production scale 从 "single-node 1B SIFT (DiskANN paper)" 推到 "**distributed 50B at Bing**"，HNSW production 上限 ~1B (Milvus segment / Qdrant single binary 内存约束) 远不及。**关键 NEW**：Microsoft Bing **已从 SPANN-style clustered partitioning 切换到 DistributedANN (Vamana-based single graph distributed)**——graph-based path 在 Bing 的 production 主导地位**重新建立** (SPANN 是 cluster-based)。NSG production 仍零。

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **不变 4** |
| NSG production | 仅 Taobao 2B | **不变** |
| Vamana production scale ceiling | DiskANN single-node 1B SIFT | **50B × 384-d at Bing (Single distributed graph)** |
| Microsoft Bing production graph algorithm | SPANN (cluster-based) was main | **DistributedANN (Vamana-based) replaces SPANN** |

### Vamana 通过 DistributedANN 进入 production scale 主导（NEW）

[per adams-2025-distributedann §1-2]

> DISTRIBUTEDANN has replaced conventional scale-out architectures for serving the Bing search engine.

→ DistributedANN 是 Vamana (DiskANN graph algorithm) 的 distributed 演化。Bing 选择 DistributedANN 替换 SPANN-style clustered partitioning 意味着**graph-based path 在最大 production case 重新主导**。Trade-off 表 (Table 1, 50B × 384-d int8, same machine footprint, 3 replicas)：
- DistributedANN: Recall@5 90.8%, p50=26ms, **6× throughput vs SPANN**
- SPANN-style: Recall@5 83.0%, p50=16ms (faster latency), 1/3 SSD

Bing 选择 graph (DistributedANN) over cluster (SPANN-style) 因为 6× throughput + 7.8pp recall + 7.5× less IO outweigh 2.9× more SSD + 1.6× slower p50 latency.

### HNSW vs NSG vs Vamana 选择决策表（updated 2026-05-11 post adams-2025）

| 工程考量 | HNSW | NSG | Vamana (DiskANN/DistributedANN) |
|---|---|---|---|
| Production OSS DBMS | 4: Milvus / Qdrant / Weaviate / Vespa | NSSG + Taobao | DiskANN (single-node) + **DistributedANN (Bing 50B distributed)** + FreshDiskANN (streaming) + Filtered-DiskANN |
| Production scale ceiling | ~1B (per single binary) | 2B (Taobao) | **50B per slice × multiple slices (Bing) = hundreds of billions** |
| Streaming insert+delete | HNSW α=1 fail → patch (HFresh/FreshVamana) | ✗ | **✓ FreshVamana α=1.2 (Vamana 原生支持)** |
| Filter-aware build | ✓ via ACORN / Filterable HNSW / Vespa Acorn-1 | ✗ | **✓ FilteredVamana / StitchedVamana** |
| GPU-native | ✓ via CAGRA | ✗ | (Vamana on GPU CAGRA-style 未独立发布) |
| Distributed single graph | ✗ | ✗ | **✓ DistributedANN (Bing production)** |
| Object storage friendly | ✗ (too many roundtrips) | ✗ | **✓ via DistributedANN KV-store-as-shared-disk** (DiskANN 本身 SSD optimized 但 DistributedANN show 跨 KV store viable) |
| Cold query 友好 | ✗ | ✗ | ✗ for traditional DiskANN; **DistributedANN compressed-vectors-in-graph 减 IO 但 latency 仍 26ms** |
| Top-level production | OSS DBMS layer | 学术 | **Microsoft Bing production main (Vamana via DistributedANN)** |

### 选择决策（updated 2026-05-11 post adams-2025）

- **Hundreds-of-billions production + distributed single graph + sublinear scaling** → **DistributedANN (Bing-validated, closed-source path)**
- **OSS vector DBMS production 主流** → **HNSW** (Milvus / Qdrant / Weaviate / Vespa)
- **Object storage primary + 多 tenant** → **Turbopuffer SPFresh** (cluster-path)
- **复杂 ranking pipeline + tensor framework** → Vespa
- **OSS Rust + 简洁** → Qdrant
- **OSS Go + AI-native primary DB** → Weaviate
- **OSS Go + 多 index_type** → Milvus
- **闭源 SaaS managed** → Pinecone
- **CPU + single-node + 简单 TopK + 静态** → NSG (学术 benchmark)
- **CPU + single-node + billion-scale + SSD** → DiskANN
- **Streaming + graph path** → FreshDiskANN (Vamana α=1.2) OR HNSW base (Weaviate HFresh preview)

NSG 仍无 production OSS DBMS；HNSW 是 RAM/NVMe-based OSS production 主流；**Vamana 通过 DistributedANN 成为 distributed production 主流 graph algorithm at Bing 50B scale**.

### MSR 谱系 5 代演化（NEW）

[per adams-2025-distributedann acks + author list]

同 MSR 团队（Qi Chen / Harsha Vardhan Simhadri 等贯穿）的 5 代演化：
1. **DiskANN 2019** (NeurIPS) — single-node graph + SSD, 1B SIFT baseline
2. **SPANN 2021** (NeurIPS) — centroid + partition, **Bing 2021-2024 主力**
3. **FreshDiskANN 2021** — streaming graph (Vamana α=1.2)
4. **SPFresh 2023** (SOSP) — streaming cluster (LIRE protocol)
5. **DistributedANN 2025** (ICML Workshop) — **Bing 2025 主力**, distributed single graph

→ MSR 在 graph (DiskANN→FreshDiskANN→DistributedANN) 和 cluster (SPANN→SPFresh) 两条 axis 并行演化，**最终 Bing production 选了 graph path 的 distributed 形态**。

### 已知盲区

- **NSG-based 主流 production DBMS**：仍未出现
- **DistributedANN streaming support**：论文 focus on serving, streaming + distributed graph 组合是 open future work
- **DistributedANN open-source 状态**：Microsoft 内部实现, 未在 microsoft/DiskANN 开源
- **DistributedANN vs FreshDiskANN/SPFresh head-to-head**: axis 不同 (distributed vs streaming), 论文无直接对比

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
