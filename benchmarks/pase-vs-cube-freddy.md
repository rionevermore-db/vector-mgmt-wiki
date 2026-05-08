---
title: PASE vs Cube / Freddy on SIFT1M / GIST1M
type: benchmark
sources: [yang-2020-pase]
related: [../systems/pase.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../topics/index-selection.md]
created: 2026-05-08
updated: 2026-05-08
---

# PASE vs Cube / Freddy on SIFT1M / GIST1M

**TL;DR**: PASE 论文 §4 的全部实验。**SIFT1M / GIST1M** 上 vs PG 内置 Cube (GiST) 与 Freddy (SQL extension)。**PASE IVFFlat build 比 Freddy 快 4-12×**（SIFT1M: 255s vs 1020s; GIST1M: 372s vs 4388s）。**HNSW marginally better than IVFFlat** in PASE: 同 recall 下 HNSW search 略快但 build 慢 20× + index 大 2× — 论文结论"HNSW slightly better than IVFFlat"。Cube 在 GIST (960-d) 不可用 (memory exhaustion)；Freddy IVFADC_PV 比 IVFADC 准但仍远低于 PASE。**论文未实验 billion-scale 与 distributed**——印证 [wang-2021-milvus Table 1] PASE 在这两轴的 ✗ 标注。[yang-2020-pase §4]

## 实验设置

[yang-2020-pase §4]

- **平台**：Linux server, **64-core 2.50 GHz Intel Xeon**, **251 GB RAM**, **1.8 TB storage**
- **数据集**：
  - **SIFT 1M**: 128-d byte vectors（[Texmex](http://corpus-texmex.irisa.fr/)）
  - **GIST 1M**: 960-d byte vectors（同上）
  - 注：**没有 billion-scale 数据集**——论文 §1 已明示 PASE 主要 million-scale
- **配置**：
  - SIFT: shared_buffers = 4 GB
  - GIST: shared_buffers = 16 GB（数据更大）
  - **single thread regime**（论文承认 production 用 multi-process）
- **对手**：
  - **Cube**（PG 8 内置 GiST extension，dim < 100 适用，论文为 GIST 重新编译以支持 960-d）
  - **Freddy**（SQL function extension based on IVFADC，含 IVFADC + IVFADC_PV 两 variant）
- **PASE 算法**：IVFFlat + HNSW 双实现
- **Metrics**：
  - **Build performance**：build time + table size + index size
  - **Search performance**：search latency + Top-1 recall (即 R1@1)

## 主结果 1：Build performance（§4.2 Tables 1-2）

### SIFT 1M

| Extension | Build time (s) | Table (MB) | Index (MB) | Parameters |
|---|---|---|---|---|
| Cube | 314 | 1116 | 2887 | NULL |
| Freddy | 1020 | 558 | **101** | kc=1000, m=8, k=100 |
| **PASE IVFFlat** | **255** | 558 | 525 | cc=1000, sr=0.01 |
| PASE HNSW | 4942 | 558 | 8333 | bnn=16, efb=200 |

### GIST 1M

| Extension | Build time (s) | Table (MB) | Index (MB) | Parameters |
|---|---|---|---|---|
| Freddy | 4388 | 3813 | **116** | kc=1000, m=16, k=100 |
| **PASE IVFFlat** | **372** | 3806 | 3912 | cc=1000, sr=0.01 |
| PASE HNSW | 20875 | 3806 | 11718 | bnn=16, efb=200 |

**关键观察**：
- **PASE IVFFlat build 全程最快**：4× over Freddy on SIFT，12× over Freddy on GIST
- **HNSW build 慢极**：SIFT 19× over IVFFlat；GIST 56× over IVFFlat（因 graph construction 需要遍历）
- **Freddy index 最小**（用 PQ 压缩）；PASE IVFFlat 存全精度更大
- **Cube 在 GIST 上 OOM**——论文未列出
- **PASE HNSW index 大**：8333 MB / 11718 MB——graph storage overhead significant

## 主结果 2：Search latency vs Recall（§4.2 Fig 15）

[yang-2020-pase Fig 15]

定性顺位（SIFT1M 上同 query time）：

| 排名 | 方法 | Top-1 recall @ 100ms |
|---|---|---|
| 1 | **PASE HNSW** (bnn=16, efb=200) | ~1.0 |
| 2 | **PASE IVFFlat** (cc=1000, sr=0.01) | ~0.99 |
| 3 | Freddy IVFADC_PV | ~0.95 |
| 4 | Freddy IVFADC | ~0.90 |
| 5 | Cube | < brute-force baseline |

**关键观察**：
- **PASE 两实现都显著优于 Freddy / Cube**——同 recall 下延迟低数倍
- **Cube 比 brute-force 还慢**——论文论证 dim>100 时 Cube 退化
- **Freddy 不稳定**：SIFT 与 GIST 上行为差异大（GIST 上 IVFADC accuracy 低）
- **HNSW vs IVFFlat in PASE**: 高 recall 区 HNSW 略胜；低 latency 区 IVFFlat 略胜——曲线交叉

## 主结果 3：HNSW vs IVFFlat in PASE（§4.3 Tables 3-4 + Fig 16-17）

测**同 PASE 内**两实现的 trade-off：

### Build vs Storage

| Algorithm | Parameters | Build time (s) | Index size (MB) |
|---|---|---|---|
| IVFFlat | cc=100, sr=0.01 | **35** | 521 |
| IVFFlat | cc=1000, sr=0.01 | 195 | 525 |
| HNSW | bnn=16, efb=80 | 2013 | 8333 |
| HNSW | bnn=16, efb=200 | 4942 | 8333 |

→ HNSW build **15-100× slower** than IVFFlat。Index size **15× larger**。

### Search performance（[Fig 16-17]）

- HNSW search 速度略胜 IVFFlat at high recall (0.99+)
- IVFFlat 收敛速度更快（参数好调）；recall 0.97 以下与 HNSW 相当
- 论文结论：**"HNSW is slightly better than IVFFlat" but **build cost is far higher**

> **wiki 解读**：在 PG 内核约束下（8 KB page），HNSW 优势没有 [Faiss](../systems/faiss.md) / [Milvus](../systems/milvus.md) 的实测大——可能因为 PG 单线程 + page-aligned constraint。

## 与 wiki 现有 benchmark 对比

| Benchmark | 系统 | 数据 | recall metric | 主要 Take |
|---|---|---|---|---|
| HNSW vs Faiss PQ on 200M SIFT | Faiss-CPU | 200M | recall@10 | HNSW 速度赢、Faiss 内存赢 |
| Milvus vs SPTAG/Vearch | Milvus 1.x | SIFT10M / Deep10M | recall + QPS | Milvus 6.4-73× faster than baselines |
| Manu vs ES/Vearch/Vald/Vespa | Manu | SIFT10M / DEEP10M | recall + QPS | Manu HNSW 双双胜 |
| ADBV vs two-step | ADBV | SIFT1B / Deep1B / AliCommodity | hybrid query latency | 3-13× faster than two-step |
| **PASE vs Cube/Freddy** | **PASE** | **SIFT1M / GIST1M** | **R1@1 + latency** | **PASE 4-12× faster build than Freddy；HNSW marginal over IVFFlat** |

→ PASE benchmark 数据规模**最小**（million-scale）——这是 PG 路线的实测上限；其他系统都到 10M+。

## 可信度评估

- **实验设计**：
  - 作者 Ant Financial / Alibaba 团队，Cube / Freddy baseline 是 standard PG 实现
  - 公开 benchmark 数据集（SIFT1M / GIST1M）
  - **single thread regime**——论文显式说明"to ensure fairness between extensions"；production 用 multi-process
- **潜在偏向**：
  1. **数据规模 1M only**——比其他 vector DBMS benchmark 小一档；不能从数据推断 billion-scale 行为
  2. **Cube 用法不公平**：Cube 设计 dim < 100，论文为 GIST 960-d 重新编译，OOM 是预期；这是"对 Cube 不利的对比"
  3. **HNSW build slow** 可能因 PG 内核单线程约束；其他系统的 HNSW build 显著快——不能直接外推 PG-HNSW 慢说"HNSW 慢"
  4. **SIFT / GIST 仅 1M**——recall 数字易达 1.0；recall plateau 区分能力差
- **复现难度**：低-中。
  - 数据公开
  - PASE 早期开源（GitHub forrest-2007/PASE）
  - PG 11 时代代码；移植到 PG 12+ 需要改动
- **场景局限**：
  - million-scale 上限——production 多 PG 实例 + 应用分片场景未实验
  - L2 + 内积；MIPS-specific 优化未涉及
  - 维度 128 / 960；现代 768 / 1024-d 未测
  - 单线程；PASE production 用 multi-process（论文未给数字）
  - **没有与 Faiss / Milvus / 其他 vector DBMS 直接对比**——只与 PG ext 比；类似 PASE 的 PG-internal context 比较

## 与其他 wiki 大规模部署对比

| 部署 | 数据 | 索引 | 类型 | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | Library | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 多机分片 | Production | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | Library + 分布式 | [douze-2024-faiss-library §7.1] |
| DiskANN @ z840 | 1B | Vamana + PQ + SSD | 算法系统 | [subramanya-2019-diskann] |
| SPANN @ Bing | 1B+ / 千亿+ | HBC + closure + SSD | 算法系统 | [chen-2021-spann] |
| Milvus 1.x SIFT1B | 1B | IVF_FLAT + HNSW etc | Vector DBMS | [wang-2021-milvus] |
| SPFresh @ Azure lsv3 | 1B | SPANN + LIRE | 算法系统 in-place | [xu-2023-spfresh] |
| Manu @ AWS m5.4xlarge | 100M（实测） | HNSW / IVF | Vector DBMS | [guo-2022-manu] |
| ADBV @ Smart City | 13B records / 30 TB | VGPQ + HNSW lambda | OLAP-extended | [wei-2020-analyticdb-v] |
| **PASE @ Ant Financial / Alipay** | **million-scale per instance**（多 PG 实例 + 应用分片到 billions production）| **IVFFlat + HNSW in PG** | **OLTP RDBMS-extended** | **本论文** |

PASE 是 wiki 已 ingest 工业部署中**单实例最小数据规模**——这是 PG 路线的代价。但作为**OLTP RDBMS-extended-vector** 路径独有，覆盖了 transaction-heavy 场景（金融 / 支付）—— 其他系统都偏 read-heavy / OLAP / vector-first。
