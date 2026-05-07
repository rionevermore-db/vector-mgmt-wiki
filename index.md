# Index

> 全 wiki 目录。每条 1 行摘要。新建 page 时同步更新本文件。

## Concepts（算法 / 数据结构）

- [HNSW](./concepts/hnsw.md) — 分层 proximity graph，O(log N) ANN，事实工业标准
- [NSW](./concepts/nsw.md) — HNSW 前作，单层 proximity graph，polylog 复杂度
- [NSG](./concepts/nsg.md) — MRNG 工程化近似，单层 + 单 entry point，Million-scale 上击败 HNSW，Taobao 已部署
- [Proximity Graph](./concepts/proximity-graph.md) — 一类基于"邻近关系"的图（k-NN / Delaunay / RNG / MSNET / MRNG），HNSW / NSW / NSG / FANNG / NNDescent 的共同基础
- [Product Quantization (PQ / IVFADC)](./concepts/product-quantization.md) — 子向量独立量化的 ANN 编码 + 倒排剪枝，Faiss IVFPQ 原型
- [ScaNN (Anisotropic VQ)](./concepts/scann.md) — Google 2020 提出的 score-aware quantization loss，把 PQ 改造为 MIPS 原生算法
- [WarpSelect](./concepts/warpselect.md) — Faiss-GPU 的 k-selection 算法，状态全在寄存器、单次扫描，55% 峰值带宽

## Systems（产品 / 工程系统）

_暂无（Faiss 库本身待 Douze 2024 论文 ingest 后建立 systems/faiss.md）。_

## Topics（跨概念主题）

- [MIPS vs L2-NN](./topics/mips-vs-l2-nn.md) — 最大内积搜索与最近邻搜索的根本差异，影响 ScaNN / HNSW / NSG / PQ 的设计与适用边界
- [GPU vs CPU ANN](./topics/gpu-vs-cpu-ann.md) — CPU 偏好图遍历、GPU 偏好 brute-force + fused k-selection；同一算法在两种硬件上最优形态不同

## Benchmarks（测评）

- [HNSW vs Faiss PQ on 200M SIFT](./benchmarks/hnsw-vs-faiss-200m-sift.md) — HNSW 论文 §5.4：HNSW 速度赢、Faiss 内存赢
- [PQ on SIFT/GIST recall + 2B SIFT](./benchmarks/pq-sift-recall.md) — PQ 论文 §V：ADC 完胜 SH/HE，IVFADC 比 ADC 快约 2×，可扩展到 2B 向量
- [NSG vs Graph ANNs on Million-Scale](./benchmarks/nsg-vs-graph-anns-million.md) — NSG 论文 §4.1：NSG 在四个百万级数据集上击败 HNSW / FANNG / KGraph 等
- [ScaNN on Glove1.2M MIPS](./benchmarks/scann-glove1.2m-mips.md) — ScaNN 论文 §5：anisotropic loss 把 Recall1@10 从 0.83 拉到 0.91，且击败 ann-benchmarks 11 个算法
- [Faiss-GPU on SIFT1B / DEEP1B / YFCC100M](./benchmarks/faiss-gpu-sift1b-deep1b.md) — Faiss-GPU 论文 §6：SIFT1B 8.5×、DEEP1B 4 GPU 抵 128 CPU 服务器、YFCC100M 35 min 构图

## Queries（高价值 query 答案存档）

_暂无。_

## Sources（原始资料速查）

详见 [`sources/README.md`](./sources/README.md)。
