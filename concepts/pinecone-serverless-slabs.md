---
title: Pinecone Serverless Slabs（Adaptive 索引架构）
type: concept
sources: [pinecone-docs]
related: [../systems/pinecone.md, ../topics/disk-vs-memory-ann.md, ../topics/in-place-vs-out-of-place-updates.md, ./pinecone-pod-based.md]
created: 2026-05-08
updated: 2026-05-08
---

# Pinecone Serverless Slabs

**TL;DR**: Pinecone 第二代核心架构。**Slab** = 不可变 file in distributed object storage；写入先进 memtable，定期 flush 为新 slab；后台 merge 小 slab 到大 slab。**关键创新**：**adaptive indexing**——不同生命周期阶段的 slab 用**不同的索引算法**——小 slab 用 fast indexing（低开销），大 slab 用 sophisticated methods（高质量）——"amortizes the cost of more expensive indexing through the lifetime of the namespace"。slab 缓存在 memory + local SSD；冷 slab 从 object storage fetch。是 LSM-style + 算法演化的混合，wiki 内首次出现"索引随生命周期演化"思路。[per sources/docs/pinecone/llms-full.txt §Architecture]

## 提出背景

Pinecone 2024 后核心 serverless 设计。针对 [pod-based](./pinecone-pod-based.md) 的两个根本痛点：

1. **静态预配置**：用户必须预估容量；空闲 pod 仍计费
2. **scaling 复杂**：垂直/水平/replica 三套机制 + collection rebuild downtime

Slab 架构借鉴：
- **LSM-tree**（memtable + 不可变文件 + 后台 merge）—— database 经典模式（与 [Milvus 1.x LSM segment](../systems/milvus.md) 同源）
- **Snowflake / S3-native warehouse 哲学**—— compute / storage 解耦，按需 elastic（与 [Milvus 2.x cloud-native 重写](../systems/milvus.md) 同代浪潮）
- **Adaptive indexing**——slab 大小决定索引方法（**Pinecone 自创**）

## 三层结构

### Layer 1：Memtable（写入缓冲）

[per sources/docs/pinecone/llms-full.txt §Architecture "Index builder"]

```
Write Request → Request Log (LSN) → Memtable (in-memory)
   ↑                                     ↓ 阈值或周期
   └──── 200 OK 同步返回 ────────       Flush
                                          ↓
                                  Slab (immutable, object storage)
```

- **每个 namespace 维护独立 memtable**（heap-style index）
- 写入持久化由 request log（write-ahead log，含 sequence number）保证
- **读取路径同时检查 memtable**——刚写入即可被搜到，无 staleness 窗口

### Layer 2：Slab（不可变 + 智能索引）

[per sources/docs/pinecone/llms-full.txt §Architecture "Object storage" + "Index builder"]

```
所有 slab 不可变 + 存对象存储（S3 / GCS / Azure Blob）

小 slab (新 flush)         大 slab (merge 后)
├─ fast indexing           ├─ sophisticated indexing
├─ 构建快                   ├─ 构建慢但 query 快
├─ 资源轻                   ├─ 资源重
└─ query 时合并所有 slab     └─ 主要 query 负载
```

**Adaptive indexing 的工程含义**：
- 新 namespace 启动时——大量小 slab，快速索引足够
- namespace 成熟后——大 slab 合并产生，更准更快索引（索引开销在 slab 生命周期内 amortize）
- 索引方法对用户**完全透明**——Pinecone 自动选

### Layer 3：Cache（query 时数据驻留）

[per sources/docs/pinecone/llms-full.txt §Architecture "Query executors"]

```
Query Executor:
  1. memory cache hit → 返回结果
  2. local SSD cache hit → 返回结果
  3. 全 miss → 从 object storage 读 slab → cache → 返回结果
```

每 slab 第一次访问 cold；后续 hot。冷启动 latency 是 serverless 模型的固有 trade-off。

## 写读路径完全解耦

[per sources/docs/pinecone/llms-full.txt §Architecture "Write path / Read path"]

```
WRITE PATH:                READ PATH:
──────────                  ────────
API gateway                 API gateway
   ↓                           ↓
Request log                 Query routers
   ↓                           ↓
Memtable                    Query executors
   ↓                           ↓ (parallel)
[Index builder] Flush        ┌─ memtable
   ↓                          ├─ slab cache (hot)
Slab in object storage  ←──── └─ object storage (cold)
```

两条路径**独立 scale**：
- 写多读少 → 多 index builder
- 读多写少 → 多 query executor
- 突然写洪 → 不影响 query latency
- 突然 query 洪 → 不影响写吞吐

这与 [pod-based](./pinecone-pod-based.md) 的"pod 同时承担读写"语义不同——后者高写时 query latency 跳。

## 与 wiki 已有概念的对比

| | Pinecone Slabs | [Milvus segment + LSM](../systems/milvus.md) | [SPANN posting list on SSD](../systems/spann.md) | [DiskANN graph + SSD](../systems/diskann.md) |
|---|---|---|---|---|
| 不可变性 | ✓（slab merge 创新文件） | ✓（segment merge） | ✗（updated in place） | ✗ |
| 存储介质 | Object storage（S3/GCS/Azure Blob）| Local FS / S3 / HDFS | NVMe SSD | NVMe SSD |
| Memtable / WAL | ✓ + LSN 持久化 | ✓ MemTable + Woodpecker WAL | ✗（静态） | ✗（静态） |
| Adaptive indexing | **✓（小 vs 大 slab 不同算法）** | ✗（segment 内 index 类型选定后冻结） | ✗ | ✗ |
| 计算 / 存储解耦 | ✓ 完全（serverless） | ✓（v2.x cloud-native） | ✗ | ✗ |
| 写读路径解耦 | ✓ | ✓ | ✗（共用） | ✗（共用） |
| 索引演化 | **生命周期内自动** | LSM merge but 索引固定 | 周期 rebuild（静态） | 周期 rebuild（静态） |

**关键差异**：Adaptive indexing 是 wiki 内**首次** 见的"同 namespace 内索引算法随时间演化"思路。其他系统（包括 [SPFresh](../systems/spfresh.md) LIRE 这种 in-place 系统）都是单一索引算法贯穿全生命周期；Pinecone 的 slab 让"早期廉价 + 晚期精确"两个目标在同一 namespace 内并存。

## Slab 与 [In-Place vs Out-of-Place Updates](../topics/in-place-vs-out-of-place-updates.md) 主题

Slab 既不是纯 in-place 也不是纯 out-of-place：
- **写入 in-place**：新数据进 memtable + 写 LSN
- **数据移动 out-of-place**：memtable flush → 新 slab；小 slab merge → 新大 slab；旧 slab GC
- **本质类似 Milvus LSM**：DBMS-style 异步整理

但有一**关键差异**——slab merge 同时**升级索引算法**（fast → sophisticated）。这是 LSM 模式 + adaptive indexing 的**叠加**——wiki 内独例。

## Open Questions

- **Adaptive indexing 的具体算法**：Pinecone docs 仅说"smaller slabs use fast indexing techniques"和"larger slabs use sophisticated methods"——**未公开具体算法**。可能是：小 = HNSW + 较小 ef，大 = HNSW + 大 ef、或 IVF + PQ；docs 未明示
- **Slab 大小阈值**：何时 trigger flush？何时 merge？多大 slab 算"大"？docs 未给参数
- **冷启动 P99 latency**：第一次 query 一个 namespace 时所有 slab 全 miss + 全从 object storage fetch——P99 多少 ms？docs 仅提"on-demand uses best-effort caching"
- **跨 region**：每 namespace 在一个 region；跨 region 复制 docs 提了但未深入
- **slab merge 的资源成本**：与 [SPFresh](../systems/spfresh.md) LIRE 的 in-place 0.4% 触发率比，Pinecone slab merge 频率 / 资源 docs 未给数字
- **vs [SPANN posting list](../systems/spann.md) on SSD**：两个都是 IVF / cluster-based + object/SSD storage，但 Pinecone slab 加 memtable + adaptive indexing。具体性能差异 wiki 未量化（无独立 benchmark）
- **算法 transparency**：用户对 slab 内部使用什么索引算法**完全无控制**——这是 Pinecone 哲学（"index_type 不暴露给用户"）vs Milvus / Faiss 哲学（"用户选 index type"）的对立。Trade-off 是简化 vs 调优能力
