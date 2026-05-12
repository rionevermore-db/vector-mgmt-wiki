---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, systems/pase.md, systems/vbase.md, topics/disk-vs-memory-ann.md]
cited-count: 4
---

# Post-snapshot (pgvector-docs): scale-tier-shifts

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 加 wiki tier-shift 一个 "deployment friction tier" 新维度**——之前 wiki tier-shift 全部基于 scale × storage × parallel 维度. pgvector ingest 揭示 **deployment friction 是 production tier-shift 决策的隐含 axis**: 小规模 + 已 Postgres workload, pgvector zero-friction "add an extension"; 大规模 + dedicated workload, separate DBMS 提供 specialized 能力但 higher operational complexity. **关键 NEW**: tier-shift 不仅是 scale, 也是 **"是否值得引入 separate vector infrastructure" decision point**——通常 ~10M-100M 之间.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| Tier-shift criteria | scale × storage × parallel × MRL prefix × sparse | **+ deployment friction (是否引入 separate vector DBMS)** |
| Tier 1 (≤1B) production main | HNSW + memory | **+ pgvector universally deployed at ≤100M scale** |
| Tier-shift "infrastructure decision" | 隐含 | **NEW: 10M-100M 之间是 "stay Postgres vs separate DBMS" decision point** |

### Tier-shift × deployment friction 新维度（NEW axis）

[per pgvector industry deployment + dedicated DBMS production]

之前 wiki tier-shift 表格主要是 dense ANN algorithm × storage media × parallel:

| Tier | Scale | Algorithm | Storage | Parallel |
|---|---|---|---|---|
| 1 | ≤1B | HNSW | memory | single-node |
| 2 | 1-10B | HNSW + quantization | memory | multi-node |
| 3 | 10-100B | SPANN / DiskANN | NVMe | routed |
| 4 | 100B-1T | DistributedANN-style | NVMe + KV store | single graph distributed |
| 5 | ≥1T | DistributedANN / Turbopuffer | KV store / object storage | namespace fanout |

**新维度: "Infrastructure decision" tier-shift**:

| Tier | "Stay Postgres" path | "Separate vector DBMS" path |
|---|---|---|
| ≤1M docs | pgvector trivial | separate overkill |
| 1-10M docs | pgvector 主流 sweet spot | separate viable but overkill |
| **10-100M docs** | **pgvector still viable (Citus + careful config)** | **separate becomes attractive** |
| 100M-1B docs | pgvector tight (assertion-based limit) | separate dominant |
| 1-100B docs | pgvector + Citus theoretically possible, no production case | separate (Milvus / Vespa / Turbopuffer) |
| ≥100B docs | pgvector not viable | DistributedANN / Turbopuffer / Vespa SPANN only |

→ **10M-100M 之间是 "infrastructure decision tier-shift" 拐点**——决定是 stay in Postgres (lower ops complexity) 还是 introduce separate vector DBMS (higher feature + scale).

### Industry deployment 数据反映此 tier-shift

[per DB-Engines 2024-2025 surveys + pgvector universal Postgres support]

**Smaller scale (≤10M docs) sweet spot**:
- pgvector deployment 占主流
- "Add an extension to existing Postgres" 是 default
- Managed Postgres (Supabase / RDS / etc.) 全部支持

**Mid scale (10M-100M docs)**:
- pgvector + separate vector DBMS 共存
- Decision driven by operational expertise, latency requirements, multimodal needs

**Large scale (>100M docs)**:
- pgvector deployment 罕见 (industry assertion-based)
- Separate vector DBMS (Milvus / Pinecone / Vespa / Turbopuffer) dominate

### Tier-shift 决策驱动（updated 2026-05-12 post pgvector-docs）

1. **Tier 1 (≤1M docs)**: pgvector default; separate DBMS overkill
2. **Tier 2 (1-10M docs)**: pgvector sweet spot; HNSW in-memory
3. **Tier 3 (10-100M docs)**: **decision point** — pgvector vs Milvus/Qdrant/Pinecone
4. **Tier 4 (100M-1B docs)**: separate DBMS dominant; pgvector tight
5. **Tier 5-7 (>1B docs)**: per previous tier-shift criteria (SPANN/DiskANN/DistributedANN/...)

### pgvector tier-shift internal (within pgvector)

[per README §Performance + Scaling]

Pgvector 内部 tier-shift:
- ≤1M docs: single instance, HNSW default
- 1-10M docs: HNSW + tune ef_search + halfvec compression
- 10-100M docs: HNSW + halfvec + binary quantization + partial indexes for filter
- 100M+ docs: Citus distributed sharding (production case rare)

### 已知盲区

- **10M-100M decision point quantification**: 不存在公开 cost / quality benchmark 在此 tier 跨 pgvector vs separate DBMS
- **pgvector + Citus 实际 production scale**: 千亿 docs 不公开
- **Tier-shift decision factor**: latency / operational expertise / cost trade-off 的 production rule-of-thumb 不公开
- **VACUUM + HNSW production cost in pgvector**: 100M+ scale write workload 不公开

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/vbase.md](../../systems/vbase.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
