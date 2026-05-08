---
title: Pinecone（Commercial SaaS Vector Database）
type: system
sources: [pinecone-docs, douze-2024-faiss-library, wang-2021-milvus]
related: [faiss.md, milvus.md, diskann.md, spann.md, spfresh.md, ../concepts/pinecone-pod-based.md, ../concepts/pinecone-serverless-slabs.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/filtered-vamana.md, ../concepts/acorn.md, ../topics/index-selection.md, ../topics/disk-vs-memory-ann.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md, ../topics/in-place-vs-out-of-place-updates.md, ../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md, ../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
created: 2026-05-08
updated: 2026-05-08
---

# Pinecone

**TL;DR**: 商业**闭源 SaaS 向量数据库**——不是 library（[Faiss](./faiss.md)）也不是开源 DBMS（[Milvus](./milvus.md)），而是 fully managed cloud service。两代架构：**Pod-based**（legacy，2025-08-18 后对新 Standard/Enterprise 客户关闭）vs **Serverless**（当前默认，含 [Slab + adaptive indexing](../concepts/pinecone-serverless-slabs.md)）。运行在 AWS / GCP / Azure；写读路径完全解耦，每路径独立 elastic scale。Single-namespace 内 [Dedicated Read Nodes](#dedicated-read-nodes-架构) 提供 provisioned hardware 替代 multi-tenant on-demand。是 wiki 内**唯一**的闭源 SaaS 系统。[per sources/docs/pinecone/llms-full.txt §Architecture]

## 与现有 wiki 系统的定位差异

[per sources/docs/pinecone/llms-full.txt §Comparison（来自 Milvus docs comparison.md）+ §Architecture]

| | [Faiss](./faiss.md) | [DiskANN](./diskann.md) | [SPANN](./spann.md) | [Milvus](./milvus.md) | [SPFresh](./spfresh.md) | **Pinecone** |
|---|---|---|---|---|---|---|
| 类型 | Library | 算法系统 | 算法系统 | 开源 DBMS | 算法系统（in-place） | **闭源 SaaS** |
| 部署模式 | embed in app | 自部署 | 自部署 | self-host / Zilliz Cloud | 自部署 | **SaaS only**（Pinecone 自营） |
| 源码可见 | ✓ | ✓ | ✓ | ✓ | ✓（基于 SPTAG） | ✗ |
| Index type 用户可见 | ✓（IVFPQ/HNSW etc） | ✓ | ✓ | ✓ | ✓ | **✗（自动选 + adaptive）** |
| Auto-elastic scaling | ✗ | ✗ | ✗ | 部分 | ✗ | **✓（serverless）** |
| Multi-region | ✗ | ✗ | ✗ | 部分 | ✗ | **✓** |
| Storage / compute 解耦 | ✗ | ✗ | ✗ | ✓（v2.x） | ✗ | **✓** |

**Pinecone 在 wiki 中的独特性**：唯一商业 SaaS、唯一 index_type 不暴露用户、唯一 fully managed multi-region。

## 架构图（Serverless）

[per sources/docs/pinecone/llms-full.txt §Architecture]

```
┌────────────────────────────────────────────────┐
│  Client (SDK: Python/Java/Node/Go/REST)        │
├────────────────────────────────────────────────┤
│  API Gateway                                   │
│   ├─ API key validation                        │
│   └─ Route to control plane / data plane       │
├────────────────────────────────────────────────┤
│  Control Plane (global)                        │
│   ├─ Project / index management                │
│   ├─ Billing                                   │
│   └─ Cross-region coordination                 │
├────────────────────────────────────────────────┤
│  Data Plane (regional, per cloud region)       │
│   ┌──────────────┐    ┌──────────────────┐     │
│   │ Write Path   │    │ Read Path        │     │
│   │ ┌──────────┐ │    │ ┌──────────────┐ │     │
│   │ │ Request  │ │    │ │ Query Routers│ │     │
│   │ │ log (LSN)│ │    │ │              │ │     │
│   │ └──────────┘ │    │ └──────────────┘ │     │
│   │       ↓      │    │       ↓          │     │
│   │ ┌──────────┐ │    │ ┌──────────────┐ │     │
│   │ │ Memtable │←┼────┼─│ Query        │ │     │
│   │ │ (heap)   │ │    │ │ Executors    │ │     │
│   │ └──────────┘ │    │ │ (slab cache  │ │     │
│   │       ↓      │    │ │  + memtable  │ │     │
│   │ Index Builder│    │ │  search)     │ │     │
│   │       ↓ flush│    │ └──────────────┘ │     │
│   └──────┼───────┘    └─────┼────────────┘     │
│          ↓                  ↑ fetch on miss    │
│       Slab (immutable)──────┘                  │
├────────────────────────────────────────────────┤
│  Object Storage                                │
│   └─ All slabs persisted (S3 / GCS / Azure)    │
└────────────────────────────────────────────────┘
```

[Slab + memtable 详见 concepts/pinecone-serverless-slabs.md]

## 数据模型

[per sources/docs/pinecone/llms-full.txt §Concepts]

### 实体层级

```
Organization              ← 计费 + 用户管理边界
  └─ Project              ← API key + Assistant scope
       └─ Index           ← schema + cloud region 绑定
            └─ Namespace  ← 查询单元，multi-tenant 隔离单位
                 └─ Record / Document
```

**关键约束**：
- 每 query 仅扫**一个** namespace（强制 isolation）
- Index 创建后 cloud region 不可改
- Project 内可多 index；不同 model embedding 用不同 index

### Document 模型（NEW，serverless 主流）

```yaml
schema:
  _id: required
  dense_vector: 0..1 个 field
  sparse_vector: 0..1 个 field
  full_text_search 标记的 string field: 最多 100 个
  filterable metadata: string / string_list / float / boolean
```

**单 document 可同时承载**：dense + sparse + full-text + 元数据 → 通过 `score_by` 选每查询的评分方式（`text` BM25 / `query_string` Lucene / `dense_vector` / `sparse_vector`）。

> **wiki 解读**：Pinecone document 模型与 [Milvus collection schema](./milvus.md) 同代——都把 vector 升级为"first-class field"，混合检索原生支持。但 Milvus 是 DBMS-style 用户控制 index_type，Pinecone 完全自动。

## 关键设计决策

### 1. Pod-based vs Serverless 双形态共存

[详见 [pinecone-pod-based.md](../concepts/pinecone-pod-based.md) + [pinecone-serverless-slabs.md](../concepts/pinecone-serverless-slabs.md)]

| | Pod-based（legacy） | Serverless（current） |
|---|---|---|
| 部署模式 | 用户预选 pod 类型/数量/大小/replica | Pinecone 自动 elastic |
| 计费 | per-minute pod time（idle 仍扣） | per RU + per write unit + storage |
| 索引算法选择 | 隐含 pod 类型决定 | **自动 adaptive**（slab 大小决定算法） |
| Scaling | 手动 vertical / horizontal / replica | 自动 |
| 状态 | 2025-08-18 起对新 Standard/Enterprise 客户关闭 | 默认 |

**Trade-off**：pod-based 给精确成本预测和性能可预测；serverless 给弹性和不需要容量规划。

### 2. 写读路径完全解耦（serverless）

[per sources/docs/pinecone/llms-full.txt §Architecture "Write path / Read path"]

写读独立 scale 是 cloud-native 设计核心。与 [Milvus 2.x cloud-native](./milvus.md)（Streaming Node + Query Node + Data Node 三角色）同思路。Pinecone 没显式公开三个 worker 角色名，但 query routers + query executors + index builder 是同样的角色拆分。

### 3. Index type 不暴露用户（**重大设计哲学**）

Pinecone 用户**不能选** HNSW vs IVFPQ vs DiskANN 等——只能选索引类型（dense / sparse / document）和元数据 schema。具体算法 + 参数完全 Pinecone 内部决定。

**Trade-off**：
- 优势：用户不需要懂 ANN 算法；Pinecone 可在背后切换更优算法不破坏 contract
- 劣势：高级用户不能 fine-tune；具体性能由 Pinecone implementation 决定，不能调优

与 [Faiss](./faiss.md) factory string + [Milvus](./milvus.md) `index_type` 显式选择是**对立设计哲学**。

### 4. Dedicated Read Nodes 架构

[per sources/docs/pinecone/llms-full.txt §Dedicated Read Nodes]

Serverless 内的"硬件保留"模式——为持续高 QPS workload 提供 provisioned read capacity：

| | On-demand（serverless 默认） | Dedicated Read Nodes |
|---|---|---|
| Read 资源 | Multi-tenant compute（共享） | **Isolated provisioned**（每 index 独占） |
| 计费 | per RU（1 RU / 1GB namespace / query, min 0.25 RU） | 固定小时费 |
| Cache | best-effort（cold start 可能） | **保证全量** in memory + local SSD |
| Rate limit | 2000 RU/sec/index 默认 | 无（仅受 CPU 限制） |
| Scaling | 自动 | 手动加 shards（storage）/ replicas（throughput） |
| Multi-namespace | ✓ | ✗（**单 namespace** only，多 namespace 后续支持） |
| 适合 | 变化 workload，多 namespace 多租户 | 持续高 QPS，单 namespace |

**Two-stage query pipeline**（仅 Dedicated Read Nodes，dense vector 上有效）：

```
Query → IVF Scanning (scan_factor 控制 partition 比例 0.5-4.0)
       ↓
   Reranking (max_candidates 控制精确距离计算量 1-100k)
       ↓
   Top-k 返回
```

- `scan_factor=4.0` 默认（96% recall）；`scan_factor=2.0` 提升 1.5× throughput 损 ~2pp recall
- `max_candidates=2500` 默认；可调到 100k 提高 recall 或调低降 latency
- API ≥ 2025-10 才支持

**Node 类型**：
- **b1**（balanced）：vector index in memory，250 GB/shard
- **t1**（performance）：vector index + projections in memory，~4× compute / ~3× cost

### 5. Multi-tenancy via Namespace（核心 isolation 机制）

[per sources/docs/pinecone/llms-full.txt §Implement multitenancy]

**namespace-per-tenant** 是 Pinecone 推荐的 multi-tenant 模式：
- 每 query 仅扫一个 namespace → 隔离 + 性能
- 大量 namespace 用 **on-demand**（best-effort cache，多 namespace OK）
- 单 namespace 高 QPS 用 **Dedicated Read Nodes**

> **wiki 解读**：[Milvus](./milvus.md) v2.6.x 4 层 multi-tenancy（database / collection / partition / partition-key）是更细粒度的；Pinecone 主要靠 namespace 单层。Trade-off：Milvus 灵活但管理复杂，Pinecone 简单但缺细粒度。

## Scale 边界

[per sources/docs/pinecone/llms-full.txt §Test Pinecone at scale + §Dedicated Read Nodes "Limits"]

| 规模 | 推荐配置 |
|---|---|
| Prototype / dev | Serverless on-demand |
| < 1M records | Serverless on-demand |
| 1M - 100M | Serverless on-demand 或 Dedicated Read Nodes（持续高 QPS） |
| 100M - 1B | **Dedicated Read Nodes**（多 shards） |
| 1B+ | **Dedicated Read Nodes 多 shards + 多 replicas**；docs 提"billion-vector datasets" but 未给具体配置 |
| 万亿 | docs 未实测覆盖 |

**Dedicated Read Nodes 资源单位**：
- Shard = 250 GB storage 单位
- Total nodes = shards × replicas
- 单 shard 通常 ~250 GB / 768-d float32 ≈ 81M vectors（推断）
- 1B 需 ~12+ shards

## 与其他系统的关系（综合）

[per sources/docs/pinecone/llms-full.txt 多处 + Milvus docs comparison]

| 系统 | 关系 |
|---|---|
| [Faiss](./faiss.md) | Pinecone 早期依赖 Faiss 作为内核 [per systems/faiss.md "其他库的关系"]，后改 Rust 重写。当前 Pinecone 内部具体使用什么算法不公开 |
| [Milvus](./milvus.md) | 平行竞品（开源 vs 闭源）。Milvus docs comparison.md 显式对比；Pinecone 不展开对比 |
| [DiskANN](./diskann.md) / [SPANN](./spann.md) | 可能在 Pinecone 内部使用（slab adaptive indexing 算法不公开），但无 source 证实 |
| [SPFresh](./spfresh.md) | 类似 in-place 思路；Pinecone slab merge 也是 LSM 风格 + adaptive indexing 加成 |

## 生产案例

[per sources/docs/pinecone/llms-full.txt + 公开材料]

Pinecone 自家不像 Milvus 公布详细客户名单。已知的 high-profile 用户（公开宣布）：Notion AI、CodiumAI、AssemblyAI、Discord 等 RAG 与 search-heavy 应用。具体规模 Pinecone 不公开。

## Open Questions

- **Pinecone 内部算法**：slab adaptive indexing 具体用什么算法？docs 仅说"fast" / "sophisticated"——可能 HNSW + ef tuning、IVF + PQ、SPANN-style、DiskANN-style 都可能；外部无法验证
- **Pinecone vs Milvus 实测对比**：Milvus docs 给 cost ranking（Pinecone vs Zilliz Cloud），但具体 latency / recall 双方都不公开
- **跨 region 复制成本与一致性**：docs 提 multi-region capability 但**未深入** failover / latency / consistency 模型
- **Slab 内部算法演化轨迹**：随 namespace 时间增长，Pinecone 是否在做"online algorithm migration"？docs 不公开
- **billion-vector benchmark**：docs 仅提"scalability to billion-vector datasets" 但未给具体配置 / latency / recall 数字（与 [Faiss trillion-scale](../benchmarks/faiss-trillion-scale.md) 相比，Pinecone 的 billion 级别公开数据极少）
- **Vendor lock-in**：闭源 SaaS 的迁移成本——切到 Milvus / Faiss self-host 的工程量，docs 不讨论
- **HIPAA + 合规** 各 cloud region 支持：docs 提"HIPAA compliance add-on"但具体细节通过 sales 流程

Cited by: 待 query 引用
