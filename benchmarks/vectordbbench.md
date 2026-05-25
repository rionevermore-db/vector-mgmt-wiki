---
title: VectorDBBench（系统级 vector DB 横测 + 公开 leaderboard）
type: benchmark
sources: [vectordbbench-docs]
related: [ann-benchmarks.md, big-ann-benchmarks.md, ../topics/ann-benchmarking-methodology.md, ../systems/milvus.md, ../systems/pinecone.md, ../systems/qdrant.md, ../systems/weaviate.md, ../systems/elasticsearch.md, ../systems/pgvector.md, ../topics/attribute-filtering.md]
created: 2026-05-21
updated: 2026-05-21
---

# VectorDBBench

**TL;DR**: Zilliz 维护的 **system-level**（整库,非裸算法）vector DB benchmark 工具 + 公开 leaderboard,横测 30+ 真实产品（Milvus / Zilliz Cloud / Pinecone / Qdrant / Weaviate / Elasticsearch / pgvector / Redis / Chroma / MongoDB / TiDB / Vespa / OceanBase …）。**wiki 内首个覆盖运维级指标的 benchmark**——load duration（ingest 耗时）、index build time、QPS、**QP$（queries-per-dollar 成本）**、p99 latency、recall;4 大 case（Capacity / Search Performance / Filtering / **Streaming insertion-under-load**）。**直接回答"Milvus 写入吞吐 / 100M 摄入耗时正不正常"这类此前 wiki 完全空白的问题**。[per sources/docs/vectordbbench/vectordbbench.md]

> **与 [ann-benchmarks](./ann-benchmarks.md) / [big-ann-benchmarks](./big-ann-benchmarks.md) 的分工**见 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md)：VDBBench = **系统级 + 成本 + 运维场景**;ann-benchmarks = 算法级 recall-QPS;big-ann = 十亿级 / 竞赛标准化。

## 实验设置

[per sources/docs/vectordbbench/vectordbbench.md]

### 被测系统（30+ client）

Milvus, Zilliz Cloud, Elasticsearch, Pinecone, Qdrant Cloud, Weaviate Cloud, PgVector, VectorChord, Redis, Chroma, CockroachDB, MongoDB, TiDB, Vespa, OceanBase, Hologres, PolarDB, Doris, Lindorm 等。

### 数据集

| Dataset | 规模 × 维度 | 用途 |
|---|---|---|
| SIFT | 500K × 128 | Small |
| GIST | 100K × 960 | 高维 |
| Cohere Wikipedia | 大规模 embeddings | 真实文本 embedding |
| OpenAI (C4) | HuggingFace C4 生成 | 真实文本 embedding |
| **LAION** | **100M × 768** | **XLarge case** |
| 自定义 | 1M-1024d / 10M-768d / 5M-1536d / 50K-1536d | 维度 / 规模扫描 |

### 硬件

标准测试 **8 core / 32 GB host**，Intel Xeon Platinum 8375C @ 2.90GHz，与被测 server 同 region（消除网络偏差）。

### 指标

- **QPS** — queries per second
- **QP$** — queries per dollar（成本效益,VDBBench 特色,面向云 SaaS）
- **Latency** — p99
- **Recall**
- **Index building time**
- **Load duration** — 数据摄入 / ingest 耗时

## 结果

### 4 大 case 类型

1. **Capacity cases**：大维 / 小维向量持续 load 直到容量满（测可装下多少 + 摄入行为）。
2. **Search Performance cases**：XLarge（100M）/ Large（10M-5M）/ Medium（1M-500K）/ Small（100K-50K）。
3. **Filtering cases**：int-based + label-based 过滤搜索（对接 [topics/attribute-filtering.md](../topics/attribute-filtering.md) 五策略框架）。
4. **Streaming cases**：**Insertion-Under-Load** —— 持续插入工作负载下评估搜索性能。

### Streaming / Insertion-Under-Load（VDBBench 1.0）

**这是直接对应"ingest 速率"问题的 case**：

- 数据集 **Cohere-10M**。
- 摄入目标 **500 rows/s = 5 个并行 producer × 100 rows/s**。
- 每摄入 **+10%** 触发一次 search 测试（serial + concurrent）。
- 测量 **p99 serial latency + max concurrent QPS @ 90% data capacity，且插入仍在进行**。
- 公开 finding：Cohere-10M 上 **Pinecone 全程 QPS+recall 高于 Elasticsearch**，Pinecone 在摄入完成后性能显著跳升。[per sources/docs/vectordbbench/vectordbbench.md]

### Timeout（侧面反映摄入规模成本）

- Medium load：2.5 h | 优化：15 min | **100M-vector load 预算：250 h**

> **wiki 解读（回应原始 query）**：100M LAION 在 VDBBench 里被归为 **XLarge**，且 load 预算给到 **250 小时量级**——印证 [systems/milvus.md] 那条 open question 与本 wiki 此前"运维吞吐盲点"判断：**百兆级摄入是小时-天量级工程,不是秒级**;真正该问的不是"裸 vec/s",而是 load duration + 持续插入下的 QPS 退化曲线（VDBBench streaming case 正是测这个）。

## 可信度评估

- **设计合理性**：同 region、统一硬件、4 类真实场景（不止纯搜索速度）、geometric-mean 跨 case 打分——比单一 recall-QPS 图更贴 production。
- **潜在偏向**：**Zilliz（Milvus 母公司）自维护**——self-benchmarking 风险。第三方 **benchANT** 维护独立 fork 可交叉验证。
- **复现难度**：低-中。开源 + 可视化 UI,无需技术背景即可跑;但 XLarge（100M）case 需大机器 + 长时间（250h 量级）。
- **数字易变**：leaderboard 随版本 / 提交滚动,**绝对 QPS/QP$ 不可冻结引用**——引用方法论稳定,引用数值须标 "as of"。详见 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md) "benchmark 可信度" 段。

## Cited by

- [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md)
- [queries/hybrid-retrieval-benchmark-landscape.md](../queries/hybrid-retrieval-benchmark-landscape.md)
