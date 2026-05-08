---
title: Pinecone Pod-Based Sharding（Legacy）
type: concept
sources: [pinecone-docs]
related: [../systems/pinecone.md, ../topics/index-selection.md, ./pinecone-serverless-slabs.md]
created: 2026-05-08
updated: 2026-05-08
---

# Pinecone Pod-Based Sharding

**TL;DR**: Pinecone 第一代索引架构（**legacy**：2025-08-18 起对新 Standard/Enterprise 客户关闭）。索引由用户预先选择的固定数量 + 类型 + 大小的 **pod**（pre-configured hardware unit）组成；scaling 是手动决策（vertical pod size doubling 或 horizontal collection rebuild + replicas）。三种 pod 类型——p1（performance）、p2（high QPS）、s1（storage-optimized）——容量与延迟权衡固定。是 wiki 已有 [Faiss IVFShards](../systems/faiss.md) / [NSG @ Taobao 32-partition](../concepts/nsg.md) 的商业 SaaS 同代变体。[per sources/docs/pinecone/llms-full.txt §Understanding pod-based indexes]

## 提出背景

Pinecone 早期（2019-2024）SaaS 部署模型——把 ANN 索引"打包"成 pre-configured hardware units 卖给用户。用户**预先**为 collection 决定：
- Pod 类型（性能 vs 容量 trade-off）
- Pod 数量（决定总容量）
- Pod 大小 x1/x2/x4/x8（线性翻倍）
- Replica 数（线性 QPS 扩展 + zone 冗余）

每 pod 静态分配硬件资源——预测性强但弹性弱。

> **legacy 状态** [per sources/docs/pinecone/llms-full.txt §Scale pod-based indexes Warning]：2025-08-18 起，新 Standard/Enterprise 客户**不能**创建 pod-based index——必须用 [serverless](./pinecone-serverless-slabs.md)（含 [Dedicated Read Nodes](../systems/pinecone.md) 选项）。已有 pod 索引继续服务但写入接近上限时不能扩。

## Pod 类型与容量

[per sources/docs/pinecone/llms-full.txt §Understanding pod-based indexes "Pod types"]

| Pod | 定位 | 单 pod 容量（768-d 向量） | QPS / latency 特征 | 备注 |
|---|---|---|---|---|
| **s1** | Storage-optimized | **5M 向量** | 较高延迟 | 大索引 + 宽松 latency |
| **p1** | Performance | 1M 向量 | <100ms 低延迟 | 最常用基准 |
| **p2** | High throughput | 1M 向量 | **<10ms（128-d, top_k<50, 200 QPS/replica）** | 不支持 sparse vector；upsert 慢（128-d 300 ups/s, 768-d 50 ups/s） |

**Pod 大小**：x1（默认）/ x2 / x4 / x8。每步 size 翻倍 storage + compute capacity。

**Pod 类型一旦选定不能改**——只能通过 collection rebuild 创新索引切换。

## 三种 scaling 操作

[per sources/docs/pinecone/llms-full.txt §Scale pod-based indexes]

### 1. Vertical scaling（pod size 翻倍）

- `x1 → x2 → x4 → x8`，单步 capacity × 2
- **10 min 完成 + 0 downtime**——读写不间断
- 不能 downscale（必须 rebuild collection）
- 触发线：index ≥ 90% fullness

### 2. Horizontal scaling A：增加 pod 数（**downtime 路径**）

- 不直接支持 in-place 加 pod
- 流程：暂停 upsert → 创 collection（snapshot）→ 从 collection 创新索引（用户指定新 pod 数 + 类型 + metadata 配置）→ 切换 URL → 删旧索引
- 优点：可换 pod 类型 + 改 metadata indexing 配置 + 增量加 pod 数（vs vertical 必须翻倍）
- 缺点：**upsert 暂停 + URL 改变** = 应用层切换成本

### 3. Horizontal scaling B：增加 replica（**0 downtime**）

- Replica 复制完整 pod 数据 + 资源
- **QPS 线性扩展**（每 replica = +1 pod QPS）——documented "if 25 QPS / replica then n replicas → 25n QPS"
- 自动跨 zone 分布（最多 3 zone）→ 多 zone 冗余
- 不增加 storage capacity（仅 throughput）

## 资源 / 计费模型

[per sources/docs/pinecone/llms-full.txt §Understanding pod-based indexes "Pod costs"]

```
total cost =
    (number of pods)      <-- 静态
  × (pod size multiplier) <-- x1/x2/x4/x8
  × (number of replicas)  <-- 线性
  × (minutes pod exists)  <-- 持续计费
  × (pod price per minute)
+ collection storage cost
```

按 **per-minute pod time** 计费，**不论 query 是否活跃**——空闲 pod 仍计费。这是 Pinecone 推到 [serverless](./pinecone-serverless-slabs.md) 的核心动机：弹性 + 用量计费。

## 与 wiki 已有概念对比

| | Pinecone Pod-based | [Faiss IVFShards](../systems/faiss.md) | [NSG @ Taobao](../concepts/nsg.md) | [Milvus 1.x reader replicas](../systems/milvus.md) |
|---|---|---|---|---|
| 部署形态 | SaaS（闭源） | Library（开源） | 内部（Alibaba） | DBMS（开源） |
| Pod / shard 抽象 | pre-configured hardware unit | 自定义切分 | 32 manual partition | reader instance |
| Scaling 颗粒 | pod size x2 / +replica / collection rebuild | 自实现 | manual rebuild per partition | reader 加副本 |
| 自动 elasticity | ✗ | ✗ | ✗ | ✓（K8s 触发） |
| 计费 | per-minute pod | n/a | n/a | per-minute resource |

## 关键 trade-off

| 优势 | 劣势 |
|---|---|
| 可预测性能（hardware 预先分配） | 弹性差（必须预估容量） |
| 简单 mental model（pod 类型 + size + 数量） | 资源浪费（idle pod 仍计费） |
| 多 zone 冗余（replica 自动分布） | scaling 操作中可能有 downtime（加 pod） |
| 精确成本估算（pod-minute × 价目） | 不能跨 region |

## 典型用法示例

[per sources/docs/pinecone/llms-full.txt §Understanding pod-based indexes "Cost example"]

> 1M × 1536-d 向量，150 QPS @ top_k=10, 1 GB collection，EU 区域：
> 选 `p1.x2 × 3 replicas + 1 GB collection`
> 月成本 ≈ \$514.54（267,840 pod-minute × \$0.0012）

## Open Questions

- **Pod-based 实际覆盖规模上限**：每 p1.x8 = 8M vec/768d；典型 deployment 多少 pod？docs 未给 production 数字
- **Performance benchmark vs serverless**：Pinecone 自家 [Test Pinecone at scale](../sources/docs/pinecone/llms-full.txt) 推荐 serverless，未给 pod vs serverless 直接对比
- **Pod-based 在 2025-08-18 后的迁移路径**：[per sources/docs/pinecone/llms-full.txt §Migrate a pod-based index to serverless] 给详细步骤但 **延迟 / cost 退化未量化**
- **不支持的功能**：sparse vector（仅 s1/p1, p2 不支持）、跨 region replication（pod-based 单 region）
- **Replica 加跨 zone**：自动分布到最多 3 zone；4+ replica 同 zone——可用性 trade-off 文档未深入

## 与 [Pinecone Serverless Slabs](./pinecone-serverless-slabs.md) 的关系

Pinecone 的两代架构形态：
- **Pod-based**：静态 pre-configured + 用户管理 scaling
- **Serverless slabs**：动态 + Pinecone 自动 scaling + slab on object storage + memtable

后者是前者的 cloud-native 重写——与 [Milvus 1.x → 2.x cloud-native 重写](../systems/milvus.md) 同样路径。两个 Pinecone 形态可在同一 Pinecone account 共存（legacy pod 索引 + 新 serverless 索引）。
