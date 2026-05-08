---
title: Woodpecker（Milvus 2.6 zero-disk WAL）
type: concept
sources: [milvus-docs]
related: [../systems/milvus.md, ./delta-consistency.md, ../topics/disk-vs-memory-ann.md]
created: 2026-05-07
updated: 2026-05-07
---

# Woodpecker

**TL;DR**: Milvus 2.6 自研的 cloud-native Write-Ahead Log，**zero-disk 设计**——所有 log 直接写到 object storage（S3 / GCS / MinIO），meta 走 etcd，**完全无本地磁盘**。取代了 Milvus 1.x / 2.0-2.5 用的 Kafka / Pulsar 外部 broker。S3 backend 实测 750 MB/s 吞吐（Kafka 130 MB/s、Pulsar 107 MB/s），但延迟 166 ms（vs Kafka 58ms / Pulsar 35ms）。两种部署：MemoryBuffer（嵌入式，200-500ms 写延迟）与 QuorumBuffer（3-replica quorum，single-digit ms）。[per sources/docs/milvus/site/en/reference/architecture/woodpecker_architecture.md]

## 提出背景

Milvus 1.x（[SIGMOD 2021 论文](../sources/papers/wang-2021-milvus.pdf)）用 Kafka/Pulsar 作 log broker；Manu (2.x) [VLDB 2022](../sources/papers/guo-2022-manu.pdf) §3.3 把 "log as data" 形式化为 backbone service，但仍依赖 Kafka/Pulsar 作消息队列——典型分布式系统设计，但带来三个问题：

1. **运维成本**：Kafka / Pulsar 自身是分布式 broker，需独立管理 disk volume / RAID / broker 故障转移
2. **本地磁盘依赖**：broker 节点需要 local disk，违背 cloud-native "stateless compute" 原则
3. **吞吐瓶颈**：Kafka/Pulsar 单 broker 节点上限 ~100 MB/s；S3 单 EC2 上限 ~1.1 GB/s——broker 成 cloud 部署的瓶颈

Milvus 2.6 用 Woodpecker 重新设计 WAL 层，**直接利用 cloud object storage 作为持久层**，broker 退化为 stateless 缓冲层。这是 Manu (2.x) "log as data" 哲学的更彻底实现——把 [Manu 2022](../systems/milvus.md) 时代的 Kafka/Pulsar 外部依赖也消除。

## 核心设计：Zero-Disk

[per sources/docs/milvus/site/en/reference/architecture/woodpecker_architecture.md]

```
┌──────────────────────────────────────────┐
│ Client（issue read/write）                │
├──────────────────────────────────────────┤
│ LogStore                                 │
│ ├─ 高速 write buffering                  │
│ ├─ 异步 upload to storage                │
│ └─ log compaction                        │
├──────────────────────────────────────────┤
│ Storage backend (S3 / GCS / MinIO / EFS) │
│ + Etcd（metadata + log state）           │
└──────────────────────────────────────────┘
```

**关键约束**：no local disk dependencies for core operations。

## 两种部署模式

### MemoryBuffer 模式（轻量 / 维护免）

- Embedded client buffer 暂存写入 → 周期性 flush 到 object storage
- Metadata 走 etcd
- **写延迟 200–500 ms**（受 S3 PutObject 周期约束）
- 适合：批量重的 workload、小规模部署、对低延迟不敏感的生产环境

### QuorumBuffer 模式（低延迟 / 高耐久）

- Client 与 **3-replica quorum** 交互
- 写入 ack 当 ≥2/3 replicas 确认（典型 single-digit ms）
- 数据异步 flush 到 cloud object storage 长期持久化
- 适合：实时响应 + 强一致 + 故障容忍的关键业务

## 性能对比

[per sources/docs/milvus/site/en/reference/architecture/woodpecker_architecture.md 性能表]

| 系统 | Kafka | Pulsar | WP MinIO | WP Local | WP S3 |
|---|---|---|---|---|---|
| Throughput | 130 MB/s | 107 MB/s | 71 MB/s | **450 MB/s** | **750 MB/s** |
| Latency | 58 ms | 35 ms | 184 ms | **1.8 ms** | 166 ms |

参照测试机存储后端的理论上限：MinIO ~110 MB/s / Local ~600-750 MB/s / S3（单 EC2）~1.1 GB/s。Woodpecker 实测达到各 backend 的 60-80%——middleware 极高效率水平。

**关键 take**：
- Local file mode：3.5× over Kafka, 4.2× over Pulsar；1.8 ms 延迟；适合单机高性能部署
- S3 mode：5.8× over Kafka, 7× over Pulsar；166 ms 延迟；high throughput batch workload
- MinIO mode：与 Kafka/Pulsar 持平，但**资源需求显著更小**

## Woodpecker 在 Milvus 架构中的位置

[per sources/docs/milvus/site/en/reference/architecture/architecture_overview.md, streaming_service.md]

```
┌─────────────────────────────────────────┐
│ Client                                  │
├─────────────────────────────────────────┤
│ Access Layer (stateless proxy)          │
├─────────────────────────────────────────┤
│ Coordinator (single active, master-     │
│   slave HA)                             │
├─────────────────────────────────────────┤
│ Worker Nodes                            │
│ ├─ Streaming Node ←──── 与 WAL 绑定 ────┐
│ ├─ Query Node                          │
│ └─ Data Node                           │
├─────────────────────────────────────────┤
│ Storage                                │
│ ├─ Meta storage (etcd)                 │
│ ├─ Object storage (S3 / MinIO)         │
│ └─ WAL storage ◄─── Woodpecker ────────┘
└─────────────────────────────────────────┘
```

WAL component 在 Streaming Node 上 exactly-one 运行，绑定的解约束由 Streaming Coordinator + 底层 WAL fencing 共同保证。

## 与 SIGMOD 论文对应物的对比

| | Milvus 1.x（SIGMOD 2021）| Milvus 2.6 + Woodpecker |
|---|---|---|
| WAL 后端 | Pulsar / Kafka 外部 broker | **Woodpecker** + S3/MinIO |
| 本地磁盘依赖 | broker 需要 | **零** |
| 吞吐上限 | Kafka 130 MB/s 类 | **750 MB/s on S3** |
| 写延迟 | ~30-60 ms | 1.8 ms (local) / 166 ms (S3) / single-digit ms (QuorumBuffer) |
| 运维 | broker 集群 + disk + replica | object storage + etcd |
| 适合 | 静态规模 + 本地 disk | **cloud-native + 弹性扩缩** |

## 与其他 cloud-native WAL 路线的对比（推断）

> [推测，wiki 未覆盖具体对比 source]
>
> Snowflake / BigQuery / Aurora 等 cloud DB 也有"WAL on object storage"思路：
>
> - **Aurora redo log shipping**：把 redo 直接发给 storage layer，但用 6-replica + 3-AZ 自定义存储（不是 S3）
> - **Snowflake 的 micro-partition 直写 S3**：但 Snowflake 是 OLAP，没有真正的 WAL
> - **Woodpecker 独特性**：直接把 OLTP 风格的 WAL 落到 commodity S3，靠 quorum buffer 弥补延迟
>
> wiki 当前未 ingest 这些系统的论文/文档，无法给精确对比。

## Open Questions

- **S3 RTT 抖动**：WP S3 mode 的 166 ms 延迟是 best-case；S3 throttling / region 跨域时 P99 可能数秒。Milvus 文档未给延迟分布数据
- **多 region / 多云部署**：跨 region 的 WAL 同步延迟、cross-region replication 文档未深入
- **成本**：S3 PUT 请求成本随 QPS 线性增长；高写吞吐下成本曲线 vs Kafka/Pulsar 自营成本未量化
- **耐久性极限**：S3 11-nines durability 已极高，但 Woodpecker 的 metadata 在 etcd——etcd 跨 region 部署的耐久语义未与 S3 协调
- **与 Aurora-style storage layer 的关系**：Aurora 把 storage 也作为 compute（log 应用在 storage 端）；Woodpecker 似乎只用 S3 作 dumb log——是否 future 演化方向？文档未讨论
- **Recovery time**：故障后 streaming node 从 Woodpecker 重放 WAL 的 RTO 上限文档未给
