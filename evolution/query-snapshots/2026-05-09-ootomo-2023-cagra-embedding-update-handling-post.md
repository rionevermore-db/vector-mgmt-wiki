---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, concepts/cagra-graph.md]
cited-count: 2
---

# Post-snapshot (ootomo-2023-cagra): embedding-update-handling

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 不解决 embedding model 升级**——与所有 14 个已 ingest source 一致。**仍是 zero coverage**。CAGRA 是 **static graph**——model 升级时**全量重 embed → 全量重 build**，但 GPU 上 build 速度极快（A100 SIFT-1M 14.5 s vs CPU HNSW 32.3 s = 2.2×），让"周期 rebuild"在 GPU 上比 CPU 更经济。**对 model migration 的影响**：(a) per-segment GPU rebuild 比 CPU 快 2.2-27×（CAGRA build），(b) 可以 segment-by-segment rolling migration（旧 model on CPU + 新 model on GPU 同时 serve），(c) GPU memory budget 严格——大 dataset migration 受限。但 **跨 model mapping function 仍 zero coverage**——14 个 ingest 后绝对 frontier 不变。

## Answer

### 与之前 ingest 的演进

| | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|
| Vector update 算法解 | + FreshDiskANN streaming insert | **不变** |
| Embedding upgrade 算法解 | 仍 zero | **仍 zero**（14 个 ingest 后均确认） |
| Rebuild 工程便利 | + per-segment rolling rebuild + streaming | **+ GPU rebuild 2.2-27× faster than CPU HNSW** |

### CAGRA build 速度对 model migration 的影响（NEW）

[per benchmarks/cagra-vs-hnsw-ggnn-ganns.md + ootomo-2023-cagra Fig 11/15]

Model 升级触发 full rebuild。CAGRA build 速度优势：

| Dataset | CAGRA build (A100) | HNSW build (CPU 64-core) | Speedup |
|---|---|---|---|
| SIFT-1M | 14.5 s | 32.3 s | 2.2× |
| GIST-1M | 28.9 s | 901.1 s | **31×** |
| GloVe-200 | 25.1 s | 172.2 s | 6.9× |
| NYTimes | 7.6 s | 226.3 s | **30×** |
| DEEP-100M | 1305.6 s | 2623.3 s | 2× |

→ **per-segment GPU rebuild 在 model migration 时比 CPU 显著快**——尤其高维数据集（GIST/NYTimes 高 30×）。这让"周级 model rolling migration" 在 GPU cluster 上变得轻松。

### CAGRA + Model migration plan（NEW）

理论上 GPU CAGRA cluster 的 segment-by-segment migration：

```
For each segment in N (parallel across GPU node):
  1. Old segment GPU CAGRA index (旧 model embeddings) 仍 serve queries
  2. 后台 re-embed segment 的 raw documents 用新 model
  3. New segment GPU CAGRA build (~ 1200s on A100 for 33M segment)
  4. Atomic swap: 切流量到 new GPU index
  5. Drop old GPU index
```

**关键工程便利**:
- GPU build 速度让单 segment migration 周期短 (10-30 min)
- 多 GPU 并行 → cluster 级 migration 几小时完成

**但 caveat**：
- GPU memory ≤80 GB → migration 期间需双倍 memory 装下旧 + 新 segment → 实际可同时 migrate ~50% segments
- Re-embedding 本身（model inference）成本不被 CAGRA 优化——是 model serving infrastructure 的事

### CAGRA + Old/new model 同时 serve（推断）

CAGRA cluster 可以多 GPU 实例**同时 serve 旧 + 新 model query**:

```
Option A: 同 GPU node, 多 instance:
   instance_old (旧 model GPU CAGRA) | port 9000
   instance_new (新 model GPU CAGRA) | port 9001
   query coordinator routes by user request

Option B: 不同 GPU nodes:
   nodes_old (旧 model) | full cluster
   nodes_new (新 model) | full cluster (parallel)
   weighted routing during transition period
```

→ GPU 路径下"old + new 双 model 同时 serve" 比 CPU + SSD 更轻 (per-instance 更小)。

### 仍是 zero coverage 的核心问题（不变）

[per 14 个 ingest 反复确认]

1. **跨 model embedding mapping function**：zero coverage
2. **Re-embedding 期间 storage 翻倍**：CAGRA 没解决（GPU memory 翻倍更昂贵）
3. **Dimension 变化必须新 index**：仍 yes (维度变 → CAGRA graph 全重 build)
4. **Query model 升级 vs data model 升级不一致**：未涉及

### 跨 ingest 累积"embedding upgrade"状态

| Ingest | 算法贡献 | 工程便利贡献 |
|---|---|---|
| 前 11 个 ingest | 0 | 0 |
| zhang-2023-vbase | 0 | 0 |
| gao-2024-rabitq | 0 | + 不需 KMeans 训练 |
| wang-2024-starling | 0 | + per-segment 滚动 rebuild |
| singh-2021-freshdiskann | 0 | + streaming insert during migration |
| **ootomo-2023-cagra** | 0 | **+ GPU rebuild 2.2-27× faster than CPU + per-segment migration shorter** |

→ 15 个 source 后 algorithm-level 解仍为 0；engineering-level 便利累积。**跨 model migration 的真正解仍未出现**——继续强化 talk 当天 live demo 那篇 2026 SIGMOD 论文 *Integrating Vector Databases across Embedding Models* 的演示价值。

### 已知盲区（仍未覆盖，15 个 ingest 后仍是绝对 frontier）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model mapping function**：仍 zero
- **GPU + CPU hybrid migration**：完全空白
- **CAGRA + 多 model concurrent serve**：理论可行未实证
- **NVIDIA RAFT model migration tooling**：未公开
- **Production embedding upgrade with CAGRA**：所有 15 ingest 都 zero coverage

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
