---
title: Faiss Trillion-scale Index（1.5T × 144-d, Meta 内部部署）
type: benchmark
sources: [douze-2024-faiss-library]
related: [../systems/faiss.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../topics/index-selection.md]
created: 2026-05-07
updated: 2026-05-07
---

# Faiss Trillion-scale Index

**TL;DR**: Faiss 论文 §7.1 描述 Meta 内部 1.5 万亿向量索引的部署。每向量压缩到 54 字节，HNSW 10M-centroids coarse quantizer，三阶段构建（2000 shard → 100 lists → 中央 mmap），83 TiB 索引分布到 20 服务器，单查询约 1 秒。[douze-2024-faiss-library §7.1]

## 实验设置

- **数据规模**：1.5 万亿向量，144 维
- **应用场景**：Meta 内部内容审核（详细未公开）
- **目标**：高召回 + 1s 级查询响应
- **存储约束**：vectors 必须压缩到 54 字节/vector（不然 1.5T × 144 × 4B = 864 TiB）

## 索引设计

### 编码方案

`PCAR72,SQ6` —— 带随机化的 PCA 降维到 72 + 6-bit scalar quantizer。

- 144d × 4B → 72d × 6/8 B = 54 字节/向量
- 总存储：1.5T × 54B ≈ 81 TiB（实测 83 TiB）

### Coarse quantizer

10M centroids 的 HNSW，用 `faiss.clustering` 的 distributed GPU k-means 训练。

- 论文判断：N=1.5T 下 K_IVF / √N ≈ 8（10M / √1.5T），低于推荐区间 15-20，但 [HNSW](../concepts/hnsw.md) coarse quantizer 让 quantization 本身极快，可以容忍更小的 K_IVF。

### 三阶段构建

1. **Shard over IDs**：把 1.5T 输入分到 2000 个独立 shard，各自 build 索引（每 shard 装得进 256 GB RAM）
2. **Shard over lists**：100 台机器分别合并 2000 索引到 100 索引（每索引 100k 倒排 list），写入分布式文件系统
3. **Load shards**：中央机器 `OnDiskInvertedLists` mmap 全部 100 索引（83 TiB）

阶段 1 + 2 是 cluster job（数百节点 × 64 核 × 256 GB RAM），阶段 3 单机 mmap。

## 结果

| 指标 | 值 |
|---|---|
| 总向量数 | **1.5 × 10¹²** |
| 单向量编码大小 | 54 字节 |
| 总索引大小 | ~83 TiB |
| Coarse centroids | 10M（HNSW） |
| 单查询时间（1 中央机器） | ~12 s |
| 单查询时间（分散到 20 中央机器） | **~1 s** |

中央机器的瓶颈是网络带宽（fetch 倒排 list），所以分散到 20 个中间服务器。[douze-2024-faiss-library §7.1 末尾]

## 与其他大规模部署对比

| 部署 | 规模 | 索引类型 | 单机 / 分布式 | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | 单 Titan X | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 分布式 | 32 分区 | [fu-2017-nsg] |
| **Faiss trillion @ Meta** | **1.5T** | **IVF + SQ6 + HNSW coarse** | **20 服务器 mmap** | **本论文** |

## 可信度评估

- **实验设计**：内部部署案例，作者直接参与；具体数据集和应用未公开（"Meta 内容审核"是泛述）
- **潜在偏向**：单一案例，无对照；论文未给召回率/精确度数字（只说"work in production"）
- **复现难度**：极高 —— 需要 1.5T 真实向量数据 + 100 节点 cluster；学术界几乎不可能复现
- **场景局限**：
  - 编码到 54 字节意味着精度有损；下游容忍度未量化
  - mmap 83 TiB 假设有快速分布式存储；网络带宽是新瓶颈
  - 1s 响应对内容审核够用；OLAP 场景未必接受

## 工程要点

1. **N=1.5T 时 K_IVF 偏离 √N 推荐**：HNSW coarse quantizer 让 K_IVF 选择更灵活
2. **PCA + SQ 是经典组合**：PCAR（带随机化的 PCA）减弱方向性；SQ6 平衡压缩与精度
3. **分阶段构建是关键**：单机 build 1.5T 不可能，shard-over-IDs + shard-over-lists 是工业模式
4. **mmap + 分布式存储替代 in-memory 索引**：内存预算超 TB 时唯一可行的存储模型

## Open Questions

- **更新策略**：1.5T 索引如果数据分布漂移，rebuild 成本极高。论文未讨论 incremental update 方案
- **网络瓶颈**：分散到 20 服务器后，跨机 fetch 倒排 list 是新瓶颈；论文未给具体网络栈优化
- **召回率**：54 字节编码下 recall 数字未公开；这是"够用就好"还是"严格保证"？

Cited by: [queries/index-architecture-global-vs-routed.md](../queries/index-architecture-global-vs-routed.md)
