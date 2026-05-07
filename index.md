# Index

> 全 wiki 目录。每条 1 行摘要。新建 page 时同步更新本文件。

## Concepts（算法 / 数据结构）

- [HNSW](./concepts/hnsw.md) — 分层 proximity graph，O(log N) ANN，事实工业标准
- [NSW](./concepts/nsw.md) — HNSW 前作，单层 proximity graph，polylog 复杂度
- [NSG](./concepts/nsg.md) — MRNG 工程化近似，单层 + 单 entry point，Million-scale 上击败 HNSW，Taobao 已部署
- [Proximity Graph](./concepts/proximity-graph.md) — 一类基于"邻近关系"的图（k-NN / Delaunay / RNG / SNG / MSNET / MRNG / Vamana），HNSW / NSW / NSG / Vamana / FANNG / NNDescent 的共同基础
- [Product Quantization (PQ / IVFADC)](./concepts/product-quantization.md) — 子向量独立量化的 ANN 编码 + 倒排剪枝，Faiss IVFPQ 原型
- [ScaNN (Anisotropic VQ)](./concepts/scann.md) — Google 2020 提出的 score-aware quantization loss，把 PQ 改造为 MIPS 原生算法
- [WarpSelect](./concepts/warpselect.md) — Faiss-GPU 的 k-selection 算法，状态全在寄存器、单次扫描，55% 峰值带宽
- [Vamana](./concepts/vamana.md) — 单层 graph + α-controlled RobustPrune + 两遍构建，DiskANN 的内存层算法

## Systems（产品 / 工程系统）

- [Faiss](./systems/faiss.md) — Meta 开源的 ANN 算法工具箱，C++17 + Python；工业事实标准；不是数据库
- [DiskANN](./systems/diskann.md) — Microsoft 开源的 SSD-resident ANN 系统，单机 64 GB RAM 跑 1B SIFT @ 98% recall；graph + SSD 路线
- [SPANN](./systems/spann.md) — Microsoft 开源的 SSD-resident ANN 系统，centroids in DRAM + posting lists on SSD；inverted file 路线；Bing 几千亿规模生产
- [Milvus](./systems/milvus.md) — Zilliz 开源的 vector DBMS（不是 library/算法系统）；建在 Faiss 之上 + LSM segment + shared-storage 分布式 + 五策略 attribute filtering + multi-vector query；SIGMOD 2021，LF AI 孵化

## Topics（跨概念主题）

- [MIPS vs L2-NN](./topics/mips-vs-l2-nn.md) — 最大内积搜索与最近邻搜索的根本差异，影响 ScaNN / HNSW / NSG / PQ 的设计与适用边界
- [GPU vs CPU ANN](./topics/gpu-vs-cpu-ann.md) — CPU 偏好图遍历、GPU 偏好 brute-force + fused k-selection；同一算法在两种硬件上最优形态不同
- [Index Selection](./topics/index-selection.md) — Faiss 决策树式索引选型：N + memory + 增量需求 + filtered search 的组合决定 index 类型
- [Disk vs Memory ANN](./topics/disk-vs-memory-ann.md) — SSD vs DRAM 的 ANN 路线；DiskANN（graph）与 SPANN（inverted file）两条 SSD 路线对比
- [Attribute Filtering](./topics/attribute-filtering.md) — 向量+属性混合查询的五策略框架（Faiss IDSelector / AnalyticDB-V cost-based / Milvus partition-based）；后者比前者快 13.7×
- [Multi-Vector Queries](./topics/multi-vector-queries.md) — 多向量 entity 的 top-k 查询；vector fusion（仅适用内积）vs iterative merging（基于 Fagin NRA，通用）

## Benchmarks（测评）

- [HNSW vs Faiss PQ on 200M SIFT](./benchmarks/hnsw-vs-faiss-200m-sift.md) — HNSW 论文 §5.4：HNSW 速度赢、Faiss 内存赢
- [PQ on SIFT/GIST recall + 2B SIFT](./benchmarks/pq-sift-recall.md) — PQ 论文 §V：ADC 完胜 SH/HE，IVFADC 比 ADC 快约 2×，可扩展到 2B 向量
- [NSG vs Graph ANNs on Million-Scale](./benchmarks/nsg-vs-graph-anns-million.md) — NSG 论文 §4.1：NSG 在四个百万级数据集上击败 HNSW / FANNG / KGraph 等
- [ScaNN on Glove1.2M MIPS](./benchmarks/scann-glove1.2m-mips.md) — ScaNN 论文 §5：anisotropic loss 把 Recall1@10 从 0.83 拉到 0.91，且击败 ann-benchmarks 11 个算法
- [Faiss-GPU on SIFT1B / DEEP1B / YFCC100M](./benchmarks/faiss-gpu-sift1b-deep1b.md) — Faiss-GPU 论文 §6：SIFT1B 8.5×、DEEP1B 4 GPU 抵 128 CPU 服务器、YFCC100M 35 min 构图
- [Faiss Trillion-scale Index](./benchmarks/faiss-trillion-scale.md) — Faiss 论文 §7.1：Meta 内部 1.5T × 144-d 索引，54 字节/向量，20 服务器 mmap 83 TiB
- [DiskANN on SIFT1B](./benchmarks/diskann-sift1b.md) — DiskANN 论文 §4：1B SIFT 1-recall@1 = 98.68% @ <5ms（同等内存下 IVFOADC+G+P plateau 62.74%）
- [SPANN vs DiskANN on Billion-scale](./benchmarks/spann-vs-diskann-billion.md) — SPANN 论文 §4：在三个 billion-scale 数据集上 SPANN 比 DiskANN 在 90% recall 时快 2×
- [Milvus vs SPTAG / Vearch / 商业系统](./benchmarks/milvus-vs-prior-sift10m-deep10m.md) — Milvus 论文 §7：6.4×–73× faster than Vearch / SPTAG / 商业 ABC；SIFT1B 单节点 + 12 节点近线性扩展；cache-aware 2.7× / AVX512 1.5× / SQ8H 系统胜 pure CPU/GPU

## Queries（高价值 query 答案存档）

- [Index Architecture: Global vs Routed (千亿/万亿)](./queries/index-architecture-global-vs-routed.md) — 工业主流走 (c) 层次路由；Meta 1.5T 部署给出具体形态。含 with vs without wiki 对比附录

## Sources（原始资料速查）

详见 [`sources/README.md`](./sources/README.md)。
