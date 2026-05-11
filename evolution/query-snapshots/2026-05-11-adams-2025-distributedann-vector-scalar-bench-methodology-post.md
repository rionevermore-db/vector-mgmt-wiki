---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/spann.md, systems/turbopuffer.md, systems/qdrant.md, systems/weaviate.md, systems/milvus.md, systems/vespa.md, topics/attribute-filtering.md]
cited-count: 8
---

# Post-snapshot (adams-2025-distributedann): vector-scalar-bench-methodology

## TL;DR (post-ingest of adams-2025)

DistributedANN paper **不深入 attribute filter** — focus on serving 50B single graph 没有 scalar-filter 维度 head-to-head. **关键 NEW**: DistributedANN 给 wiki **fair benchmark framework 一个重要新 dimension**——同 scale 内**"single graph distributed vs partitioned graph distributed"**的公平对比。Table 1 是 wiki 内**首次完整 production-scale head-to-head**：同 hardware footprint, 同 dataset (50B Bing web slice), 同 replicas (3), 9 metrics (recall@5, recall@200, p50, p99, SSD, memory, IO, network BW, throughput)——可作为 wiki 内 fair benchmark **best-in-class 范本**。

## Answer

### Wiki 当前覆盖的 7 vendor filter 实现（更新计数）

| Vendor | Filter 实现 | Source |
|---|---|---|
| Milvus | 5 strategies | wang-2021-milvus |
| Qdrant | Filterable HNSW + ACORN fallback + Tenant/Principal Index | qdrant-docs |
| Weaviate | ACORN + correlation optimization | weaviate-docs |
| Vespa | 3-mode (pre/post/Acorn-1) by YQL planner | vespa-docs |
| Pinecone | filter + slab adaptive (不公开) | pinecone-docs |
| Turbopuffer | Native filtering with SPFresh clustering-hierarchy aware | turbopuffer-docs |
| **DistributedANN (NEW)** | **Paper 未深入 filter; focus on serving 50B single graph** | adams-2025-distributedann |

DistributedANN paper 不直接 contribute filter 工程; 但 paper Table 1 fair benchmark methodology 是 wiki 内 **best-in-class 公平对比范本**.

### DistributedANN Table 1 作为 fair benchmark 范本（NEW）

[per adams-2025-distributedann Table 1, §4]

**Fair benchmark criteria DistributedANN paper 实践得最完整**:
1. **Same dataset**: 50B × 384-d int8 Bing web index slice
2. **Same hardware footprint**: 3 replicas, same SKU mix
3. **Same workload**: sampled web search queries
4. **Multiple metrics, not just one**: recall@5 + recall@200 + p50 + p99 + SSD + memory + IO + network BW + throughput
5. **Multiple selectivity points** (Figure 4 Pareto frontier): 
   - DistributedANN: H from 4 to 8, BW = 32i for i 3-6
   - Clustered: N = {20, 25, 30, 40, 50, 60}, IO per cluster M = 32i for i 2-6
6. **Production environment, not synthetic**: actual Bing host SKU 256-768 GiB RAM / 5-10 TiB SSD / 32-64 cores / 40 Gbps
7. **Reliability degradation curve**: Table 2 shows recall vs availability 100%-96%

**Pitfall list (wiki 内 fair benchmark 常见错误)**:
- Single selectivity point (DistributedANN Figure 4 用 Pareto frontier 避免)
- Same-metric optimization (DistributedANN 给 9 metrics 综合视图)
- Synthetic workload (DistributedANN 用真实 Bing search queries)
- Hardware different (DistributedANN 严格 same SKU 但 increase storage per host)

### Vector-scalar fair benchmark 框架（updated 2026-05-11 post adams-2025）

**Tier 1: Capability inventory**:
1. Filter 实现 (per vendor)
2. Selectivity sweep ∈ {0.01%, 0.1%, 1%, 10%, 50%, 90%, 99%}
3. Attribute cardinality ∈ {10, 1K, 100K, 10M, 100M}
4. Correlation: vector ↔ attribute 正/负/独立 三种
5. Filter complexity: simple / complex / range / glob/regex
6. **Workload diversity**: not just one fixed query set

**Tier 2: 实测协议 (DistributedANN-style)**:
- **Pareto frontier**: 不是 single config, 是参数 sweep + best frontier
- **Multi-metric**: recall + latency + throughput + IO + storage + memory + BW
- **Production environment**: 真实 SKU + 真实 workload
- **Reliability/availability**: 不仅 100% available 数字, 还 graceful degradation curve
- **Storage vs compute trade-off explicit**: e.g., DistributedANN 接受 2.9× SSD 换 6× throughput——这种 trade-off 必须 explicit, 不是单 latency 数字

**Tier 3: Pitfall avoidance**:
- 一 selectivity 点 → 多 selectivity sweep
- 单 latency metric → multi-metric trade-off table
- Synthetic workload → production-like dataset
- Cold/warm 分离 (Turbopuffer 必须分报)
- 同 recall threshold (ef / nprobe 同等校准)

### 已知盲区

- **DistributedANN attribute filter**: paper 不深入; Bing production 是否 native filter integration? unknown
- **DistributedANN vs OSS DBMS filter head-to-head**: 跨 OSS / closed 路径 fair benchmark 不存在
- **6 OSS DBMS Table 1-style head-to-head**: 仍无公开 multi-metric fair benchmark
- **Turbopuffer native filtering vs DistributedANN distributed filter**: 两闭源 SaaS / production 系统都不公开足够细节

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/vespa.md](../../systems/vespa.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
