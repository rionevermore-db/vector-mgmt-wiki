---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [systems/analyticdb-v.md, concepts/vgpq.md, benchmarks/analyticdb-v-vs-twostep.md, systems/milvus.md, topics/disk-vs-memory-ann.md, topics/attribute-filtering.md]
cited-count: 6
---

# Post-snapshot (wei-2020-analyticdb-v): giga-scale-sharding

## TL;DR (delta from guo-2022-manu post)

**新增 OLAP-extended-vector 路径作为参照**：[ADBV](../../systems/analyticdb-v.md) 的 production case study 已实测 **13B records / 30 TB / 70 节点**——是 wiki 已 ingest 工业部署中第二大（仅次于 Meta 1.5T）。Lambda streaming HNSW + batching VGPQ 双索引的设计与 Milvus / Manu 三层架构同代但**用两种不同算法分担 streaming/batching 角色**。给定 16 节点私有云：ADBV-style 是**SQL 友好的另一条路线**，但需要 OLAP DB 内核（Pangu / Fuxi / AnalyticDB）支撑——开源用户难直接复刻。

## Answer

### 新增 OLAP-extended-vector 路径作为对照（NEW）

[per systems/analyticdb-v.md "与 wiki 现有系统的定位差异"]

| 路径 | 起点 | 代表 | 私有云 16 节点适用性 |
|---|---|---|---|
| Library | ANN 库 | Faiss | 自实现胶合层；不推荐 |
| 算法系统 | ANN 算法 | DiskANN, SPANN, SPFresh | 灵活但工程量大 |
| Vector-first DBMS | Vector-first | **Milvus / Pinecone** | 直接部署 K8s |
| **OLAP-extended-vector** | **关系 OLAP 加 vector** | **ADBV (闭源 SaaS), Vespa, ES vector** | **复刻难（依赖 OLAP DB 内核）** |

→ 对私有云部署，仍优先 **Milvus 2.x / SPFresh** 路径（开源可控）。ADBV 对私有云的价值是**架构启发**：lambda streaming + batching 用两种算法分工的设计可借鉴。

### ADBV 13B 实测的工程细节（NEW）

[per benchmarks/analyticdb-v-vs-twostep.md "结果 7"]

**Smart City 车辆违章检测**（Alibaba Cloud, 70 nodes, 13B records / 30 TB）：
- 视频流 → 帧抽取 → vehicle detection → embedding → INSERT
- Hybrid query: timestamp / location / camera + 视觉相似
- 延迟从 hundreds of seconds → ms

**对 16 节点私有云的推论**：
- 70 节点撑 13B records ≈ 单节点 ~185M records
- 16 节点 × 185M ≈ 3B records——千亿距离 33×
- 千亿 768-d 推断需要 ~500 节点 ADBV-style 部署——超 16 节点预算
- **千亿规模仍只 Manu / SPANN 路线接近实证可达**

### Lambda 框架对周级 update 的工程指导（NEW）

[per systems/analyticdb-v.md "Lambda 三层框架"]

ADBV streaming HNSW + batching VGPQ：
- **新数据进 streaming HNSW**（实时；resource 重）
- **周期 async merge 到 batching VGPQ**（offline 重建；不阻塞 query）
- **Serving layer merge 两层结果**

→ 对 16 节点私有云的"周级全量更新"约束：
- 不需要 batch 全量切换（这是 ADBV 论文的论证）
- "周级"在 ADBV 框架下变成"streaming layer 的 HNSW 大小阈值控制 merge 频率"
- merge 期间旧 batching 仍服务，新 batching 后台重建——**与 SPFresh in-place 不同思路**（SPFresh 增量改 index；ADBV 异步重建批量索引）

### 与之前 ingest 的 ADBV 引用关系（NEW）

之前 wiki 内 ADBV 已被多处引用但未直接 ingest：
- topics/attribute-filtering.md "Strategy D cost-based" 指 ADBV
- benchmarks/milvus-vs-prior-sift10m-deep10m.md 提及 ADBV
- wang-2021-milvus Table 1 把 ADBV 列竞品

→ 本 ingest **填补了 ADBV 自身论文的源头**。Strategy D 不再"听 Milvus 说"，可直接读 wei-2020-analyticdb-v §5。

### 已知盲区

- **ADBV 万亿规模实测**：论文实测仅到 1B；Smart City production 13B 是间接证据
- **Pangu 闭源** → 私有云用户难直接复刻 ADBV 全栈
- **ADBV vs Milvus / Pinecone 实测对比**：双方都不公开
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo

## Cited Pages

- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [benchmarks/analyticdb-v-vs-twostep.md](../../benchmarks/analyticdb-v-vs-twostep.md)
- [systems/milvus.md](../../systems/milvus.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
