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
- [Woodpecker](./concepts/woodpecker.md) — Milvus 2.6 自研 zero-disk WAL；S3 上 750 MB/s 吞吐（5.8× over Kafka）；MemoryBuffer / QuorumBuffer 双部署模式
- [LIRE](./concepts/lire.md) — SPFresh 的 Lightweight Incremental REbalancing 协议；2 必要条件 + 5 操作 + cascading 收敛证明；仅 0.4% 插入触发 rebalance
- [Pinecone Pod-Based Sharding](./concepts/pinecone-pod-based.md) — Pinecone 第一代（legacy 2025-08 关闭）：p1/p2/s1 pod 类型 + x1/x2/x4/x8 size + replica 线性 QPS
- [Pinecone Serverless Slabs](./concepts/pinecone-serverless-slabs.md) — Pinecone 第二代核心：slab on object storage + memtable LSM-style + **adaptive indexing**（小 slab fast / 大 slab sophisticated，wiki 内首次"索引随生命周期演化"）
- [Delta Consistency](./concepts/delta-consistency.md) — Manu (Milvus 2.x) 形式化的 tunable bounded staleness 一致性模型；strong / eventual 是 τ=0/∞ 的特例；wiki 内首个 vector DBMS 一致性形式化
- [Manu SSD-Aware Indexing](./concepts/manu-ssd-hierarchical-kmeans.md) — Manu §4.4 hierarchical k-means + LSH-style 多次复制（4-8×）+ 4KB SSD-aligned blocks；NeurIPS 2021 BigANN 冠军方案，比 baseline same QPS recall +60%
- [VGPQ](./concepts/vgpq.md) — AnalyticDB-V 的 IVFPQ successor；Voronoi diagram 上用 neighbor midpoints 切 subcells 几何剪枝；same index size + -10% build time + 全程优于 IVFPQ on SIFT1B/Deep1B/AliCommodity
- [FilteredVamana / StitchedVamana](./concepts/filtered-vamana.md) — Filtered-DiskANN 的 filter-aware graph 算法；首次把 label 信息 baked-in 到 graph 构造本身（不只 search 步骤过滤）；Microsoft 广告 A/B test +35-49% production gain
- [ACORN](./concepts/acorn.md) — Stanford 2024 SIGMOD predicate-agnostic HNSW 改造（ACORN-γ + ACORN-1）；首个支持 unbounded predicate set + 任意 operator（regex/contains/between/OR）；25M LAION >1000× over baselines
- [Relaxed Monotonicity](./concepts/relaxed-monotonicity.md) — VBASE OSDI 2023 形式化的 vector + relational 索引共享性质；两阶段遍历模式（Phase 1 接近 → Phase 2 离开）；让 vector index 与 B-tree 用同一套 Volcano iterator engine——绕开 TopK speculation 的理论基础
- [RaBitQ](./concepts/rabitq.md) — NTU Singapore SIGMOD 2024 首个 unbiased + sharp error bound 的 quantization；D-bit string + 随机正交矩阵旋转 hypercube vertices codebook；O(1/√D) 渐近最优；error-bound rerank 无需 K' 调参——quantizer 层对 K' 预测问题的 dual 攻击（与 VBASE iterator 范式正交可叠加）；3× 快于 PQ LUT
- [Block Shuffling](./concepts/block-shuffling.md) — Starling SIGMOD 2024 形式化的 disk graph index 数据布局问题；Theorem 4.1 NP-hard + 不存在多项式时间近似算法；3 个启发式 BNP/BNF/BNS 把 OR(G) 从 DiskANN ≈0 提升到 0.34-0.87；Starling 默认 BNF 占总 build 9.5% 时间
- [FreshVamana](./concepts/freshvamana.md) — FreshDiskANN 2021 提出的 Vamana streaming 变体；首个支持 graph-based incremental insert + delete + recall 不退化的算法；α-RNG property (α=1.2) 是 fresh-ANNS 必要条件——HNSW/NSG 因隐式 α=1 仍未解；50 cycles × 5%/10%/50% change 后 recall 95%+ 稳定；build 比 static Vamana 1.48-1.83× 快
- [CAGRA Graph](./concepts/cagra-graph.md) — NVIDIA ICDE 2024 首个 GPU-native proximity graph；fixed out-degree + non-hierarchical + directional + rank-based reordering（不需 distance 重算 → 1.9× faster, DEEP-100M 唯一可行）；search 4 个 GPU 优化（warp splitting + forgettable hash + 1-bit parented + dual single/multi-CTA）；与 [HNSW/NSG/Vamana] CPU graph 算法是不同 hardware path 的对偶

## Systems（产品 / 工程系统）

- [Faiss](./systems/faiss.md) — Meta 开源的 ANN 算法工具箱，C++17 + Python；工业事实标准；不是数据库
- [DiskANN](./systems/diskann.md) — Microsoft 开源的 SSD-resident ANN 系统，单机 64 GB RAM 跑 1B SIFT @ 98% recall；graph + SSD 路线
- [SPANN](./systems/spann.md) — Microsoft 开源的 SSD-resident ANN 系统，centroids in DRAM + posting lists on SSD；inverted file 路线；Bing 几千亿规模生产
- [Milvus](./systems/milvus.md) — Zilliz 开源的 vector DBMS（不是 library/算法系统）；建在 Faiss 之上 + LSM segment + shared-storage 分布式 + 五策略 attribute filtering + multi-vector query；SIGMOD 2021，LF AI 孵化
- [SPFresh](./systems/spfresh.md) — Microsoft 在 SPANN 之上加 LIRE 协议，**首个 billion-scale in-place 增量更新**系统；100 days × 1% daily update 持续 10 GB + 2 cores（DiskANN rebuild 需 1100 GB + 32 cores × 2 天）；SOSP 2023
- [Pinecone](./systems/pinecone.md) — wiki 内**唯一**商业闭源 SaaS；两代架构（pod-based legacy + serverless slabs）；On-demand vs Dedicated Read Nodes；index_type 不暴露给用户（adaptive 自动选）
- [AnalyticDB-V](./systems/analyticdb-v.md) — Alibaba 的 OLAP-extended-vector 系统（不是 vector-first）；SQL native hybrid query + lambda streaming/batching + 4 plan CBO + VGPQ；13B records / 30 TB Smart City production
- [PASE](./systems/pase.md) — Ant Financial 的 PostgreSQL ANN extension；首个直接在 PG kernel 注册 ANN index type 的方案（IVFFlat + HNSW）；OLTP RDBMS-extended-vector 路径；与 ADBV 同 ecosystem 但 host DB 是 OLTP；million-scale per instance（Ant Financial / Alipay 生产）
- [VBASE](./systems/vbase.md) — Microsoft Research OSDI 2023 PG 扩展；**首个 iterator-model 路径**（vs 其他全部 TopK-based）；基于 [Relaxed Monotonicity](./concepts/relaxed-monotonicity.md) 绕开 K' 预测；Q4-Q6 multi-column TopK 比 Milvus 快 200-300×，Q8 vector Join 比 PG 快 7900×；学术原型，~2000 LOC + <200 LOC per index
- [Starling](./systems/starling.md) — Zilliz/Milvus 团队 SIGMOD 2024 segment-level disk-resident graph framework；首个把 vector DBMS segment 约束（~2GB RAM + ~10GB disk）作为 design first principle；3 大贡献（block shuffling + in-memory navigation graph + block search）；ANNS 2× 快于 DiskANN，RS 43.9× 快；framework 兼容 Vamana/NSG/HNSW；§8 future work 集成 Milvus
- [FreshDiskANN](./systems/freshdiskann.md) — Microsoft Research 2021 首个 graph-based billion-scale streaming ANN 系统；LTI on SSD + TempIndex in DRAM + StreamingMerge two-pass 合并；800M SIFT sustained 1800+1800 inserts/deletes/sec @ 95+% recall；StreamingMerge 比 DiskANN 全 rebuild 5.25× 快（15832s vs 83140s）；vs PLSH 25× 少机器；is the graph-path counterpart to SPFresh's cluster-path
- [Qdrant](./systems/qdrant.md) — Qdrant Solutions GmbH 2021 Rust 实现的开源 vector DBMS（Apache-2.0），OSS + Cloud + Hybrid + Private + Edge 五 SKU；**Filterable HNSW with extra edges** filter-aware build (HNSW base, 与 FilteredVamana Vamana base 平行)；v1.16.0 集成 ACORN algorithm 作 fallback（**wiki 内 ACORN frontier 关闭**——首次 production deployment）；多 quantization 变体（Scalar / Binary / 1.5/2-bit / Asymmetric / PQ）；Collection Aliases 是 wiki 内首个明确 model migration production tool；Raft + sharding；**仅 HNSW 单 index** 与 Milvus 多 index_type 对比
- [Weaviate](./systems/weaviate.md) — Weaviate B.V. 2019 Go 实现的 OSS vector DBMS（BSD-3-Clause），定位 **"AI-native primary database"**；HNSW + RQ8 quantization default + HFresh preview (HNSW base streaming)；**第二个 ACORN production case**（继 Qdrant 之后）+ "positive/negative correlation optimization"；first-class hybrid search (BlockMaxWAND BM25 + vector) + 完整 inverted index 套件（Roaring bitmaps + bit-sliced range bitmaps + LSM）；**首个 vector DBMS vendor 显式集成 agent stack**（Query Agent turnkey RAG + Engram agent memory preview）；built-in `text2vec-weaviate` embeddings + multi-tenancy + RBAC first-class
- [Vespa](./systems/vespa.md) — Yahoo! 2003 内部开发 / 2017 开源 (Apache-2.0)，C++ + Java 实现 search + recommendation + personalization engine；**唯一来自 web search engine lineage 的 wiki 系统**；**SPANN production 第二实证**（继 Microsoft Bing，OSS 路径）；**ACORN-1 第三 production case**（继 Qdrant + Weaviate, **ACORN frontier 完全闭合**）；**Streaming Search**: no-index per-user brute-force, 45 bytes/doc, billions/node（与所有 ANN 系统哲学相反）；**4-phase ranking pipeline** + first-class tensor framework (dense/sparse/mixed) + ONNX/XGBoost/LightGBM/TF native；Application Package atomic deployment + Stateless container/Stateful content separation
- [CAGRA / NVIDIA RAPIDS RAFT](./systems/cagra.md) — NVIDIA ICDE 2024 首个 GPU-native graph-based ANN system；NVIDIA RAPIDS RAFT library 核心实现；Milvus v2.6.x GPU_CAGRA 索引基于此；build 比 HNSW (CPU 64-core) 2.2-27× 快，large-batch search 33-77× 快，single-query 3.4-53× 快；vs GGNN/GANNS (前作 GPU graph) 3.8-8.8× 快
- [Turbopuffer](./systems/turbopuffer.md) — Turbopuffer Inc. 闭源 commercial SaaS（Rust）；**wiki 内第二个 closed SaaS**（继 Pinecone 之后）；**object-storage-native 架构**——S3/GCS 是唯一 stateful 依赖，compute 节点完全 stateless（query + indexing 双 compute-compute 分离自动伸缩）；**SPFresh 首个 OSS-known production deployment**（SOSP 2023 Microsoft Research → 2026 commercial SaaS production，仅 3 年，**SPFresh frontier 关闭**）；LSM tree natively on object storage；3-tier cache (Memory + NVMe SSD + Object Storage)；WAL on object storage 提供 strong consistency by default；3.5T+ docs / 13PB+ total / 100B+ vectors queryable / 100M+ namespaces / 10M+ writes/s / 25k+ QPS production observed；continuous recall monitoring 90-100% recall@10；April 2026 namespace pinning（reserved compute per namespace）；20 公开 region (AWS+GCP) + BYOC (AWS/GCP/Azure)

## Topics（跨概念主题）

- [MIPS vs L2-NN](./topics/mips-vs-l2-nn.md) — 最大内积搜索与最近邻搜索的根本差异，影响 ScaNN / HNSW / NSG / PQ 的设计与适用边界
- [GPU vs CPU ANN](./topics/gpu-vs-cpu-ann.md) — CPU 偏好图遍历、GPU 偏好 brute-force + fused k-selection；同一算法在两种硬件上最优形态不同
- [Index Selection](./topics/index-selection.md) — Faiss 决策树式索引选型：N + memory + 增量需求 + filtered search 的组合决定 index 类型
- [Disk vs Memory ANN](./topics/disk-vs-memory-ann.md) — SSD vs DRAM 的 ANN 路线；DiskANN（graph）与 SPANN（inverted file）两条 SSD 路线对比
- [Attribute Filtering](./topics/attribute-filtering.md) — 向量+属性混合查询的五策略框架（Faiss IDSelector / AnalyticDB-V cost-based / Milvus partition-based）；后者比前者快 13.7×
- [Multi-Vector Queries](./topics/multi-vector-queries.md) — 多向量 entity 的 top-k 查询；vector fusion（仅适用内积）vs iterative merging（基于 Fagin NRA，通用）
- [In-Place vs Out-of-Place Updates](./topics/in-place-vs-out-of-place-updates.md) — 向量索引更新策略；周期 rebuild（DiskANN/Faiss/Milvus）vs in-place 增量（SPFresh LIRE）；graph-based 在 in-place 仍开放
- [TopK 接口 vs Iterator Model](./topics/topk-vs-iterator-model.md) — vector index 集成范式之争；TopK + K' 预测（Milvus / ADBV / PASE / Pinecone / Elasticsearch）vs Iterator + RM（VBASE 唯一）；Q4-Q8 上后者比前者快 100-7900×
- [Vector Range Query](./topics/vector-range-query.md) — 按距离阈值返回（distance ≤ r）；与 TopK 平行的查询模式；TopK 接口下 K_LARGE 难选；Iterator + RM 自然支持；VBASE Q7 是 wiki 内首个原生支持系统

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
- [SPFresh vs DiskANN / SPANN+ on 100-Day Update](./benchmarks/spfresh-vs-diskann-spann-update.md) — SPFresh 论文 §5：100 days × 1% daily update 模拟，SPFresh P99.9 平均 2.41× lower than DiskANN，5.30× lower memory；1B stress test 饱和 NVMe 400K IOPS
- [Manu vs Elasticsearch / Vearch / Vald / Vespa](./benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md) — Manu (Milvus 2.x) VLDB 2022 §5：HNSW + IVF-FLAT 在 SIFT10M / DEEP10M 上系统击败四个开源 vector engine baseline；vs Milvus 1.x 在 4k QPS insertion 下 search latency 不抖动（dedicated index node 设计的关键收益）
- [AnalyticDB-V vs Two-step + VGPQ vs IVFPQ](./benchmarks/analyticdb-v-vs-twostep.md) — ADBV 论文 §6：vs "AnalyticDB + 独立 ANN engine" 两步式方案 3-13× 快；VGPQ 在 SIFT1B/Deep1B/AliCommodity 全程优于 IVFPQ；4 plan CBO 自动选最优；13B records production scale
- [PASE vs Cube / Freddy](./benchmarks/pase-vs-cube-freddy.md) — PASE 论文 §4：SIFT1M / GIST1M 上 PASE IVFFlat build 比 Freddy 4-12× 快；PASE HNSW marginal over IVFFlat（recall 高但 build 慢 20×）；Cube 在 dim>100 不可用
- [Filtered-DiskANN vs Milvus / Faiss / NHQ](./benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md) — WWW 2023 §5-6：FilteredVamana / StitchedVamana 比 Milvus / Faiss-IVF / NHQ 快 5-10× QPS @ 90% recall on Microsoft 真实数据；广告 A/B test +34.61% clicks / +48.95% revenue (P=0.009-0.03)
- [ACORN vs Filtered-DiskANN / NHQ / Milvus](./benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md) — SIGMOD 2024 §7：4 datasets (LCPS + HCPS) + 25M LAION scale；ACORN-γ LCPS 上 2-10× over FilteredVamana / NHQ；HCPS 30-1000× over baselines；25M LAION >1000× over next best
- [VBASE 8-query on Recipe1M](./benchmarks/vbase-8queries-recipe1m.md) — OSDI 2023 §5：Recipe1M 330K + Tag 10K extension，8 query 类型（Q1-Q8）；VBASE Q4-Q6 multi-column TopK 比 Milvus 快 200-300×，Q7 range filter 唯一原生，Q8 vector Join 比 PG 快 7900×；selectivity sampling rate 0.001 q-error <1.1
- [RaBitQ vs PQ/OPQ/LSQ on 6 Datasets](./benchmarks/rabitq-vs-pq-opq-lsq-6datasets.md) — SIGMOD 2024 §5：6 dataset (MSong/SIFT/DEEP/GIST/Word2Vec/Image)；RaBitQ 用一半 code length 仍 dominate PQ/OPQ/LSQ time-accuracy 曲线；MSong/Word2Vec PQ avg rel error >100% RaBitQ <40%；ε₀=1.9 + B_q=4 cross all datasets 无调参；6/6 dominate HNSW
- [Starling vs DiskANN / SPANN on Segment](./benchmarks/starling-vs-diskann-spann-on-segment.md) — SIGMOD 2024 §6：4 dataset (BIGANN 33M / DEEP 11M / SSNPP 16M / Text2image 5M) on Milvus segment 配置 (2GB RAM + 10GB disk)；ANNS 2× DiskANN，RS 43.9× DiskANN（98% 低 latency），>10× SPANN on Text2image；Starling-Vamana/NSG/HNSW 全 2× 各自 baseline；BIGANN 1B 用 31 segment 跑通
- [FreshDiskANN Streaming on 800M SIFT](./benchmarks/freshdiskann-streaming-sift800m.md) — arXiv 2021 §6：单机 96 thread + 3.2 TB NVMe + 128 GB RAM；800M SIFT week-long steady-state 1800+1800 inserts/deletes/sec @ 95+% recall；StreamingMerge 5.25× faster than DiskANN full rebuild；burst 40K inserts/sec；FreshVamana α=1.2 vs α=1 in 50 cycles 5%/10%/50% change rate
- [CAGRA vs HNSW / GGNN / GANNS](./benchmarks/cagra-vs-hnsw-ggnn-ganns.md) — ICDE 2024 §V：7 datasets (SIFT/GIST/GloVe/NYTimes/DEEP-1M/10M/100M) on DGX A100；CAGRA build 2.2-27× faster than HNSW；large-batch search 33-77× faster；single-query 3.4-53× faster；vs GGNN/GANNS GPU baselines 1.0-31× faster；rank-based reordering 1.9× faster + DEEP-100M 唯一可行；FP16 mode +30% throughput

## Queries（高价值 query 答案存档）

- [Index Architecture: Global vs Routed (千亿/万亿)](./queries/index-architecture-global-vs-routed.md) — 工业主流走 (c) 层次路由；Meta 1.5T 部署给出具体形态。含 with vs without wiki 对比附录

## Sources（原始资料速查）

详见 [`sources/README.md`](./sources/README.md)。
