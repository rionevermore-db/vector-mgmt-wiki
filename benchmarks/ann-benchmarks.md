---
title: ann-benchmarks（算法级 recall-QPS Pareto 标准）
type: benchmark
sources: [ann-benchmarks-docs]
related: [vectordbbench.md, big-ann-benchmarks.md, ../topics/ann-benchmarking-methodology.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../systems/faiss.md, ../topics/gpu-vs-cpu-ann.md]
created: 2026-05-21
updated: 2026-05-21
---

# ann-benchmarks

**TL;DR**: Erik Bernhardsson 等维护的 **algorithm-level** ANN benchmark,是学术界 / 工业界引用最广的"裸算法 recall-QPS"事实标准。40+ 实现（HNSW/hnswlib、Annoy、NGT、DiskANN/Vamana、FAISS、PyNNDescent、pgvector、Milvus、Qdrant…）× 15 数据集（SIFT 128d、GIST 960d、GloVe 25-200d、DEEP1B-10M、NYTimes…）。**强制 CPU-only + single-threaded** 以隔离算法效率,结果画成 **recall-vs-QPS Pareto frontier**。刻意**不做 billion-scale**（推给 [big-ann-benchmarks](./big-ann-benchmarks.md)）。结果 "as of April 2025"。[per sources/docs/ann-benchmarks/ann-benchmarks.md]

> **定位**：与 [VectorDBBench](./vectordbbench.md)（系统级 + 成本 + 运维）和 [big-ann-benchmarks](./big-ann-benchmarks.md)（十亿级竞赛）的三方分工见 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md)。

## 实验设置

[per sources/docs/ann-benchmarks/ann-benchmarks.md]

### 评测对象（40+ 实现）

- Tree-based：Annoy, KDTree, BallTree, KGraph
- Graph-based：[HNSW](../concepts/hnsw.md), hnswlib, NGT, DiskANN, [Vamana](../concepts/vamana.md)
- LSH：FLANN, LSHForest, NearPy, RPForest
- ML frameworks：scikit-learn, [FAISS](../systems/faiss.md), PyNNDescent
- Vector DB：Milvus, Qdrant, Weaviate, Elasticsearch, OpenSearch
- DB extension：pgvector, pgvecto.rs, RediSearch

### 数据集（15 个，million-scale）

| Dataset | Dim | Train / Test | Metric |
|---|---|---|---|
| DEEP1B (10M slice) | 96 | 9.99M / 10K | Angular |
| SIFT | 128 | 1M / 10K | Euclidean |
| Fashion-MNIST | 784 | 60K / 10K | Euclidean |
| GloVe (4 变体) | 25–200 | 1.18M / 10K | Angular |
| GIST | 960 | 1M / 1K | Euclidean |
| NYTimes | 256 | 290K / 10K | Angular |

全部带 **top-100 nearest neighbors ground truth**。

### 方法论

- 硬件：AWS **r6i.16xlarge**，`--parallelism 31`，**关闭超线程**。
- **single-CPU / single-threaded 默认**（`--batch` 可开 batch mode）——隔离算法本身,排除多线程 / 系统层差异。
- Docker 容器化保证可复现;train/test + query 强制分离。

### 指标

Recall（vs top-100 GT）、QPS（single-query throughput）、build/index time、index size（隐式）。

## 结果

- 核心呈现 = **recall-QPS（precision-throughput）Pareto frontier 图**：只画前沿点,**忽略被支配的配置**。
- 6 张代表图（glove-100-angular、SIFT、Fashion-MNIST 等）。
- 维度 100-1000「challenging but realistic」。

> **wiki 解读**：ann-benchmarks 是 wiki 内多篇论文 benchmark 的"上游方法论"——[benchmarks/nsg-vs-graph-anns-million.md](./nsg-vs-graph-anns-million.md)、[benchmarks/cagra-vs-hnsw-ggnn-ganns.md](./cagra-vs-hnsw-ggnn-ganns.md) 等都以 ann-benchmarks 风格的 recall-QPS 曲线汇报。它定义了"算法级公平对比"的事实范式:固定数据集 + 固定硬件 + 单线程 + Pareto 前沿。

## 可信度评估

- **设计合理性**：single-threaded + CPU-only 是优点（隔离算法）也是局限（**不反映 GPU / 多线程 / 分布式 production**——见 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md)）。
- **偏向**：社区中立（非厂商维护）,但 vendor 可提交自家最优参数,**调参努力不对称**是已知 caveat。
- **复现难度**：低——明确以"easy to replicate"为设计目标。
- **边界**：million-scale 上限;billion-scale、filter、streaming、成本均不覆盖（分别由 big-ann / VDBBench 补）。

## Cited by

- queries/（待 query 引用时补）
