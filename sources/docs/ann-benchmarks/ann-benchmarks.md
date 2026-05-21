# ann-benchmarks — captured methodology (WebFetch summary, 2026-05-21)

> 来源：`github.com/erikbern/ann-benchmarks` README + `ann-benchmarks.com`。结果 "as of April 2025"。

## 定位

**Algorithm-level**（隔离算法效率），非整库。40+ 实现：

- Tree-based：Annoy, KDTree, BallTree, KGraph
- Graph-based：HNSW, hnswlib, NGT, DiskANN, Vamana
- LSH：FLANN, LSHForest, NearPy, RPForest
- ML frameworks：scikit-learn, FAISS, PyNNDescent
- Vector DB：Milvus, Qdrant, Weaviate, Elasticsearch, OpenSearch
- DB extension：pgvector, pgvecto.rs, RediSearch

## 数据集（15 个）

| Dataset | Dim | Train / Test | Metric |
|---|---|---|---|
| DEEP1B (10M slice) | 96 | 9.99M / 10K | Angular |
| SIFT | 128 | 1M / 10K | Euclidean |
| Fashion-MNIST | 784 | 60K / 10K | Euclidean |
| GloVe (4 变体) | 25–200 | 1.18M / 10K | Angular |
| GIST | 960 | 1M / 1K | Euclidean |
| NYTimes | 256 | 290K / 10K | Angular |

全部带 **top-100 nearest neighbors ground truth**。

## 方法论

- 硬件：AWS **r6i.16xlarge**，`--parallelism 31`，**关闭超线程**。
- **single-CPU / single-threaded 默认**（`--batch` 可开 batch mode）。
- Docker 容器化保证可复现。
- train/test 集 + query points 强制分离。
- sparse set-similarity 数据集以 sorted integer array 传入。

## 指标

- **Recall**（vs top-100 GT）
- **QPS**（single-query throughput）
- Build / index time（偏好不"极端昂贵建索引"的实现）
- Index size（隐式）

## 结果呈现

- recall-QPS（precision-throughput）**Pareto frontier 图**——只画前沿点,忽略被支配配置。
- 6 张图（glove-100-angular, SIFT, Fashion-MNIST 等）。

## 区别 / 边界

- 算法级评估,**强制 CPU-only + single-threaded** 以隔离算法效率。
- **不含 billion-scale**（推给 big-ann-benchmarks）。
- 维度 100–1000「challenging but realistic」。
- 极重可复现性。
