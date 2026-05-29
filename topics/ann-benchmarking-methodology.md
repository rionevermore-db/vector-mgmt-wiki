---
title: ANN Benchmarking 方法论（算法级 vs 系统级 vs 竞赛级）
type: topic
sources: [vectordbbench-docs, ann-benchmarks-docs, bigann-benchmarks-docs]
related: [../benchmarks/vectordbbench.md, ../benchmarks/ann-benchmarks.md, ../benchmarks/big-ann-benchmarks.md, ../benchmarks/mteb-massive-text-embedding-benchmark.md, ../benchmarks/beir-heterogeneous-zero-shot-ir.md, ../topics/attribute-filtering.md, ../topics/multimodal-embedding-retrieval.md, ../topics/gpu-vs-cpu-ann.md, ../systems/milvus.md]
created: 2026-05-21
updated: 2026-05-21
---

# ANN Benchmarking 方法论

**TL;DR**: "哪个向量检索方案更好"没有单一答案,因为 benchmark 分**三个互不替代的抽象层**:**算法级**（[ann-benchmarks](../benchmarks/ann-benchmarks.md)：裸算法 recall-QPS，CPU 单线程，million-scale）、**系统级**（[VectorDBBench](../benchmarks/vectordbbench.md)：整库横测，含 load duration / QP$ 成本 / 持续插入，真实 SaaS）、**竞赛级**（[big-ann-benchmarks](../benchmarks/big-ann-benchmarks.md)：标准化硬件 + 私有 query set，十亿级 + filter/sparse/OOD/streaming track）。选错层会得出误导结论:用算法级 recall-QPS 评估生产选型会漏掉成本与摄入退化;用厂商自测 leaderboard 评估算法本身会被调参不对称污染。

## 问题陈述

向量检索的"性能"是多维的——recall、QPS、p99 latency、build time、**load/ingest duration**、内存、磁盘、**成本（$/query）**、动态更新下的稳定性、filter / 多模态下的退化。没有任何单一 benchmark 同时公平覆盖全部。三个主流 benchmark 各自占据一个抽象层,**互补而非竞争**:

- 问"**HNSW 还是 NSG 算法更快**" → 算法级（ann-benchmarks）
- 问"**生产上选 Milvus 还是 Pinecone、贵不贵、摄入扛不扛得住**" → 系统级（VectorDBBench）
- 问"**十亿级 + filter/sparse/OOD 谁的 recall 上限高**" → 竞赛级（big-ann）

## 相关概念

- 算法层：[HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md) / [PQ](../concepts/product-quantization.md) / [Vamana](../concepts/vamana.md) —— ann-benchmarks 的评测对象
- 系统层：[Milvus](../systems/milvus.md) / [Pinecone](../systems/pinecone.md) / [Qdrant](../systems/qdrant.md) 等 —— VectorDBBench 的评测对象
- 场景层：[attribute filtering 五策略](../topics/attribute-filtering.md)（filter case/track 的理论框架）、[multimodal retrieval](../topics/multimodal-embedding-retrieval.md)（OOD/cross-modal track）、[GPU vs CPU](../topics/gpu-vs-cpu-ann.md)（ann-benchmarks 单线程 CPU 局限）
- 邻域 benchmark：[MTEB](../benchmarks/mteb-massive-text-embedding-benchmark.md)（embedding model 选型）/ [BEIR](../benchmarks/beir-heterogeneous-zero-shot-ir.md)（IR 系统 zero-shot）—— 评的是 embedding/检索质量,不是 ANN 索引性能,与本三方正交

## 工业方案对比

| 维度 | [ann-benchmarks](../benchmarks/ann-benchmarks.md) | [VectorDBBench](../benchmarks/vectordbbench.md) | [big-ann-benchmarks](../benchmarks/big-ann-benchmarks.md) |
|---|---|---|---|
| 抽象层 | **算法 / 库** | **整库系统 / 产品** | **算法 @ 标准化硬件（竞赛）** |
| 规模 | million（≤10M） | small → **XLarge 100M** | **billion（1B）** + 10M practical |
| 评测对象 | 40+ 算法实现 | 30+ vector DB 产品 | 竞赛参赛系统 |
| 硬件 | AWS r6i.16xlarge，CPU 单线程 | 8c/32G host（同 region） | Azure 标准 VM（F32s/L8s/D8lds） |
| 核心指标 | recall-QPS Pareto | + **load duration / QP$ 成本 / 持续插入 QPS** | **recall@k at fixed throughput** |
| 场景 | 纯搜索 | + filter / streaming / capacity | + filter / sparse / OOD / streaming track |
| 成本维度 | ✗ | **✓ QP$** | T3 有 cost/query |
| 防作弊 | 中（vendor 可调参） | 弱（**厂商自维护**风险） | **强（私有 query set + 标准硬件）** |
| 维护方 | 社区（Erik Bernhardsson 等） | **Zilliz（Milvus 母公司）** | NeurIPS workshop（Harsha Simhadri 等） |

## benchmark 可信度（"benchmarks lie" 的三个陷阱）

1. **调参努力不对称**：vendor 给自家产品调到最优、对手用默认参数。wiki 内实例——[ACORN 论文](../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md) 自承 "Milvus baseline 用 single-thread + 默认参数,production 可能高很多"。
2. **厂商自测偏向**：VectorDBBench 由 Zilliz 维护;benchANT 维护独立 fork 用于第三方交叉验证。另一典型——[LanceDB Enterprise 自家 benchmark 页](../systems/lancedb.md)只给自己的 latency(1M×1536d vector P99 35ms 等),**无 recall / 无对比系统 / 无硬件规格**:vendor benchmark 的常见信息缺口(报喜不报忧)。**同源延伸——doc 摘要也会"lie"**:LanceDB 首轮 ingest 据 docs 摘要断言"first-class GPU index build",深化抓一手 indexing 文档却无法印证(见 [systems/lancedb.md §4](../systems/lancedb.md))——摘要 ≠ 一手,断言须可追到具体页。
3. **leaderboard 数字易变**：VDBBench leaderboard 随版本 / 提交滚动,**绝对 QPS/QP$ 不可冻结引用**——引用方法论稳定,引用数值须标 "as of"。big-ann 的私有 query set + 标准化硬件是对这三个陷阱最强的对冲。

## Open Questions

- **空间 / geo 检索 benchmark 完全空白**：三个 benchmark 均无空间 track;呼应 [multimodal-embedding-retrieval.md](../topics/multimodal-embedding-retrieval.md) 记录的"空间能力严重不对称、无 head-to-head benchmark"——三模（vector+scalar+spatial）公平横测仍无 source。**注**:spatial **index/algorithm 能力** 2026-02 起已 Vespa + LanceDB(R-Tree)两家有 [per systems/lancedb.md §H],但 **benchmark 仍空白**——这是 wiki 内 "algorithm-成熟先于 benchmark-成熟" 的典型 case。
- **端到端 ingest throughput 的标准口径缺失**：VDBBench 有 load duration + streaming case,但"vec/s 摄入率"受 batch / bulk_insert vs row insert / 维度 / 是否并发建索引 强烈影响,无统一报告口径——这正是本 wiki 此前回答"Milvus 摄入速率正不正常"时反复标"未覆盖"的根因。
- **GPU / 分布式公平对比**：ann-benchmarks 强制 CPU 单线程,big-ann T3 才容 GPU,VDBBench 看整库黑盒——**没有 benchmark 公平隔离 GPU-native（[CAGRA](../systems/cagra.md)）vs CPU graph 的算法贡献**。
- **三方结果互译**：同一算法在算法级 Pareto 漂亮、系统级被运维开销吃掉、竞赛级被标准硬件拉平——三层结果如何互相换算尚无方法论。

Cited by: [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md), [queries/hybrid-retrieval-benchmark-landscape.md](../queries/hybrid-retrieval-benchmark-landscape.md)
