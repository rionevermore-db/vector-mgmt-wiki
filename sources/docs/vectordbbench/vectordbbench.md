# VectorDBBench — captured methodology (WebFetch summary, 2026-05-21)

> 来源：`github.com/zilliztech/VectorDBBench` README + VDBBench 1.0 发布说明。Layer-1 citation 锚点;**数字随 leaderboard 滚动**。

## 定位

System-level（整库）benchmark，**对比真实 vector DB 而非裸算法**。30+ supported client：
Milvus, Zilliz Cloud, Elasticsearch, Pinecone, Qdrant Cloud, Weaviate Cloud, PgVector, VectorChord,
Redis, Chroma, CockroachDB, MongoDB, TiDB, Vespa, OceanBase, Hologres, PolarDB, Doris, Lindorm 等。

## 数据集

- SIFT 500K × 128
- GIST 100K × 960
- Cohere Wikipedia embeddings（大规模）
- OpenAI-generated（来自 HuggingFace C4）
- **LAION 100M × 768**（XLarge case）
- 自定义：1M-1024d / 10M-768d / 5M-1536d / 50K-1536d

## 硬件

标准测试：8 core / 32 GB host，Intel Xeon Platinum 8375C @ 2.90GHz，与被测 server 同 region。

## 指标

- QPS（queries per second）
- **QP$（queries per dollar — 成本效益,VDBBench 特色）**
- Latency（p99）
- Recall
- Index building time
- **Load duration（数据摄入时间 / ingest 耗时）**

## 4 大 case 类型

1. **Capacity cases**：大维 / 小维向量持续 load 直到容量满。
2. **Search Performance cases**：XLarge（100M）/ Large（10M-5M）/ Medium（1M-500K）/ Small（100K-50K）。
3. **Filtering cases**：int-based + label-based 过滤搜索性能。
4. **Streaming cases**：**Insertion-Under-Load** —— 持续插入工作负载下评估搜索性能。

### Streaming / Insertion-Under-Load 细节（VDBBench 1.0）

- 数据集 Cohere-10M。
- **摄入目标 500 rows/s = 5 个并行 producer × 100 rows/s**。
- 每摄入 +10% 数据触发一次 search 测试（serial + concurrent 两种）。
- 测量：**p99 serial search latency + max concurrent QPS @ 90% data capacity，且插入仍在进行**。
- 公开 finding：Cohere-10M streaming 测试中 **Pinecone 全程 QPS+recall 高于 Elasticsearch**，Pinecone 在摄入完成后性能显著提升。

## Timeout（侧面反映规模成本）

- Medium 数据集 load：2.5 小时
- 优化：15 分钟
- **100M-vector load：250 小时预算**

## 与 ann-benchmarks 的区别

- 成本维度（QP$）面向云服务
- 真实场景多样性：filtering / streaming / capacity（超越纯搜索速度）
- 可视化 UI，无需技术背景即可复现
- "贴近真实 production 环境"
- leaderboard 用 geometric-mean 跨多 metric / 多 case 打分
- **system-level（整库）而非 algorithm-level**

## 可信度注意

- Zilliz（Milvus 母公司）维护——self-benchmarking 潜在偏向，需交叉验证。
- leaderboard 数字随版本滚动，不可冻结引用。
- benchANT 维护了一个独立 fork（第三方复现）。
