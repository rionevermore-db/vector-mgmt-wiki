---
title: Faiss（Library）
type: system
sources: [douze-2024-faiss-library, johnson-2017-faiss-gpu]
related: [../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/scann.md, ../concepts/warpselect.md, ../concepts/vamana.md, diskann.md, spann.md, milvus.md, spfresh.md, ../topics/index-selection.md, ../topics/gpu-vs-cpu-ann.md, ../topics/disk-vs-memory-ann.md, ../topics/attribute-filtering.md, ../topics/in-place-vs-out-of-place-updates.md, ../benchmarks/faiss-trillion-scale.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md]
created: 2026-05-07
updated: 2026-05-07
---

# Faiss

**TL;DR**: Facebook AI Research 开源的 ANN 算法工具箱，C++17 核心 + Python wrapper。**不是数据库**（无并发写、无 sharding 管理、无查询优化），只是 ANN 算法的实现集。Zilliz、Pinecone、Milvus 都依赖它作为核心引擎。9 年（2017 → 2024）累计 6M 下载、37k GitHub stars、[GPU 论文](../benchmarks/faiss-gpu-sift1b-deep1b.md) 5200+ 引用。[douze-2024-faiss-library §1]

## 架构图

```
┌──────────────────────────────────────────┐
│  C# / Rust / 其他 binding                │
├──────────────────────────────────────────┤
│  Python wrapper (SWIG)                   │
│  + contrib (datasets, benchmark, ...)    │
├──────────────────────────────────────────┤
│  C++17 core                              │
│  ├─ CPU indexes (IndexIVF, IndexHNSW...) │
│  └─ GPU add-on (GpuIndexIVFPQ...)        │
├──────────────────────────────────────────┤
│  CUDA + BLAS (MKL/openblas) + numpy      │
└──────────────────────────────────────────┘
```

[douze-2024-faiss-library Fig 9]

## 数据流 / 控制流

Index 是 Faiss 的中心抽象。所有 index 共享同一组方法：

- `train(x)` —— 用 x 训练（k-means 等）
- `add(x)` —— 加入向量
- `add_with_ids(x, I)` —— 带 63-bit 任意 ID 的 add
- `search(x, k)` —— 查 k-NN
- `range_search(x, ε)` —— 距离半径内查询
- `remove_ids(I)` —— 移除（部分 index 支持）
- `reconstruct_batch(I)` —— 反编码

[douze-2024-faiss-library §A.7]

## 关键设计决策

### 1. 库 vs 数据库的分界（§1）

Faiss **明确不做**：

- feature extraction（不嵌入原始数据）
- service（不做 HTTP / RPC）
- database（无并发写、无负载均衡、无 sharding 管理、无 transaction、无 query optimization）

**Trade-off**：把数据库语义留给上层（Milvus、Vespa、Pinecone 自己实现）；Faiss 只保证算法层面的正确性与性能。

### 2. 三轴 Pareto 框架（§3）

Faiss 把所有指标显式暴露：

- **Active constraints**：speed / memory / accuracy 是用户配置选项
- **Pareto pruning**（§3.4）：基于参数 monotonicity 的剪枝。论文示例：5808 参数组合 → 398 实测 → 87 Pareto 最优。

**Trade-off**：算法选型从拍脑袋变成可工程化，代价是用户必须知道自己的"active constraints"。

### 3. Index factory string（§A.2）

字符串描述 index 组合：

```
"PCA160,IVF20000_HNSW,PQ20x10,RFlat"
```

依次：PCA 降维到 160 → IVF 20000 lists（coarse quantizer 是 HNSW）→ PQ 20 sub-quantizer × 10 bit → 精确距离 reranking。

**Trade-off**：极强组合表达力 vs 字符串本身可读性差、调试困难。

### 4. 公开 struct 字段，零封装（§A.1）

所有 C++ 类是 `struct`，字段全 public。**Trade-off**：用户可深度 hack（callback 子类化 `ResultHandler` / `InvertedLists` / `IDSelector`），代价是 ABI 不稳定。当前 C++17 兼容（晚于业界主流）。

### 5. C++ 核心 + SWIG Python（§A.2）

SWIG 自动生成 Python 包装。**Trade-off**：迭代快但绑定层粗糙；C API 单独维护用于 Rust / Java 等。

## Index 家族（CPU）

```
IndexFlatCodes              ← 顺序存储压缩向量
├─ IndexFlat                ← 不压缩
├─ IndexPQ                  ← Product Quantizer
├─ IndexScalarQuantizer     ← SQ8 / SQ6 / SQ4 / SQfp16 / SQbf16
├─ IndexAdditiveQuantizer   ← additive quantizer 家族
│  ├─ IndexResidualQuantizer (RQ)
│  ├─ IndexLocalSearchQuantizer (LSQ)
│  ├─ IndexProductResidualQuantizer (PRQ)
│  └─ IndexProductLocalSearchQuantizer (PLSQ)
└─ IndexLSH / IndexLattice / IndexNeuralNetCodec

IndexIVF                    ← Inverted file
├─ IndexIVFFlat / IVFPQ / IVFScalarQuantizer
├─ IndexIVFAdditiveQuantizer (RQ/LSQ variants)
└─ IndexIVFFastScan         ← SIMD 4-bit layout（借鉴 ScaNN）

IndexHNSW                   ← Graph
├─ IndexHNSWFlat / IndexHNSWScalarQuantizer
IndexNSG                    ← Graph

IndexFastScan               ← 独立 fast-scan
IndexPreTransform           ← PCA / OPQ / ITQ wrapper
IndexRefine                 ← 后置精确 reranking
IndexIDMap                  ← 任意 ID 映射
IndexShards / IndexReplicas ← 分片 / 复制
```

[douze-2024-faiss-library Fig 11]

## Scale 边界

按 N 决定的实操选择（[douze-2024-faiss-library Fig 10] 决策树，详见 [topics/index-selection.md](../topics/index-selection.md)）：

| N | 推荐索引 | 备注 |
|---|---|---|
| < 10k | `IndexFlat`（brute force） | 无需索引 |
| 10k–1M | `NSG,Flat` / `HNSW,Flat` | 内存富余、graph 最快 |
| 1M–10M | `IVF16k,Flat` | flat coarse quantizer |
| 10M–100M | `IVF64k_HNSW,...` | HNSW-as-coarse-quantizer |
| 100M–1B | `IVF256k_HNSW,...` | 仍内存内 |
| 1B+ | `IVF1M_HNSW,PQ/SQ` | 必须压缩 |
| **1.5T**（[trillion-scale 案例](../benchmarks/faiss-trillion-scale.md)） | `PCAR72,SQ6` + 10M HNSW coarse + 分布式 mmap | [douze-2024-faiss-library §7.1] |

## 与其他库的关系（§5.5）

| 库 | 关系 | 备注 |
|---|---|---|
| [ScaNN](../concepts/scann.md)（Google） | Faiss 借鉴其 4-bit FastScan SIMD layout | 反向：ScaNN 是 IVFPQ + 自家 anisotropic loss |
| [DiskANN](./diskann.md)（Microsoft） | 平行竞品 | 磁盘原生（[Vamana](../concepts/vamana.md) graph + SSD）；Faiss 主要内存。详见 [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md) |
| [SPANN](./spann.md)（Microsoft / SPTAG） | 平行竞品 | 磁盘原生 IVF（centroids 内存 + posting list SSD）；Bing 几千亿规模生产。Faiss IVFPQ 的 SSD 化对应物 |
| HNSWlib / nmslib | [HNSW](../concepts/hnsw.md) 参考实现 | Faiss `IndexHNSW` 从此分叉 |
| **[Milvus](./milvus.md)** | **建在 Faiss 之上的 vector DBMS** | 内核 wrap Faiss IVF/PQ/HNSW，但补齐 Faiss 明确不做的功能：dynamic data (LSM)、distributed (shared-storage)、attribute filtering、multi-vector query、cache-aware/SIMD/SQ8H 工程改造。是 Faiss §1 "库 vs 数据库" 论点的工业 DBMS 对应物 [per wang-2021-milvus]。Knowhere 库 wrap Faiss 是 1.x 路线；Milvus 2.0+ 是 cloud-native 重写 |
| Pinecone | 早期依赖 Faiss，后改 Rust 重写 | |
| Weaviate | 复合检索引擎，含 Faiss 作为可选 | |

## 生产案例

- **Meta 内部**：trillion-scale 内容审核索引（详见 [Faiss Trillion-scale](../benchmarks/faiss-trillion-scale.md)）；SeamlessM4T 双语挖掘；1.3B 图像 deduplication（[douze-2024-faiss-library §7.3]）
- **Atlas / REPLUG / RA-DIT / kNN-LM**：retrieval-augmented LLM（§7.2）
- **Zilliz / Milvus / DeepSeek**：作者团队中 Zilliz、DeepSeek 直接是 Faiss 工业用户

## Open Questions

- **数据分布漂移下的退化**：§6.1 引用 Baranchuk 2023 提出 explicit updates，但 Faiss 当前 IVF / PQ 的 codebook 一旦 train 就冻结，long-running 索引如何 graceful 重新训练？**[SPFresh](./spfresh.md) [xu-2023-spfresh] 给出 cluster-based 一侧的成功答案**：在 [SPANN](./spann.md) 上加 [LIRE](../concepts/lire.md) 协议持续维持 NPA 性质，1B 索引 100 天 1% daily update 持续仅 10 GB memory + 2 cores。该思路理论上可以迁移到 Faiss IVFPQ（centroid 同样可 incremental rebalance），但 Faiss 当前不集成。详见 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)
- **真正的 graph 增量更新**：[HNSW](../concepts/hnsw.md) 支持 add 但不支持 suppression / mutation；[NSG](../concepts/nsg.md) 不支持任何增量。`FreshDiskANN` 是工程方向。
- **Out-of-distribution queries**：§5.2 提及 OOD-DiskANN、Filtered-DiskANN 是 frontier；Faiss 当前不直接支持。
- **GPU graph 索引**：§A.3 末尾明示 CAGRA 是 emerging direction；Faiss-GPU 当前只有 IVF 类。
- **库 vs 数据库的边界何时模糊**：[Milvus](./milvus.md) / Vespa 已经把 Faiss 包成数据库；上下游融合到什么程度时 Faiss 应该收回部分功能？[wang-2021-milvus] 给出 DBMS 一侧的具体回答：cache-aware partition、runtime SIMD hooking、SQ8H hybrid CPU/GPU、LSM segment、shared-storage 分布式、五策略 attribute filtering、multi-vector query 都属"Faiss 不做、上层补齐"

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md)
