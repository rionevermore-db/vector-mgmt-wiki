---
title: Delta Consistency（可调一致性）
type: concept
sources: [guo-2022-manu]
related: [../systems/milvus.md, ../concepts/woodpecker.md, ../topics/in-place-vs-out-of-place-updates.md]
created: 2026-05-08
updated: 2026-05-08
---

# Delta Consistency

**TL;DR**: Manu (= [Milvus](../systems/milvus.md) 2.x 学术论文) 提出的形式化**有界过期一致性模型**——介于 strong 与 eventual 之间。用户指定 tolerable staleness `τ`（virtual time 单位）；query 必须等到 subscriber 已消费的最新 time-tick `L_s` 与 query 自身 LSN `L_r` 满足 `L_r - L_s < τ` 才执行。**Strong consistency = δ=0；eventual consistency = δ=∞**——两者均为特例。论文称"first to support delta consistency in a vector database"。这是 vector DBMS 的核心 tunable performance-consistency trade-off。[guo-2022-manu §3.4]

## 提出背景

[guo-2022-manu §1, §2 设计目标]

传统关系数据库支持两种极端：
- **Strong consistency**：每读必看到最新写——延迟高、throughput 低
- **Eventual consistency**：读可能看到任意旧值——throughput 高但语义弱

Manu 论文论证：vector DB 应用**几乎都不需要复杂事务**，但对一致性需求**多样化**：
- 推荐系统：新用户行为**几秒延迟可接受**，等同步会伤体验
- 安全/反作弊：新黑名单 vector 必须**立即可见**
- 视频检索：新上传视频**几分钟内可见**即可

→ 单一一致性级别不够；需要 user-tunable 的中间态。

## 核心定义

[guo-2022-manu §3.4]

```
delta consistency parameter τ (用户指定，单位 = virtual time)

每 query 在 access layer 收到时分配 LSN L_r（write request 也分配 LSN）
每 log subscriber（query node）当前消费到的最新 time-tick = L_s

query 执行条件：
   L_r − L_s < τ
否则 query node 等待下一个 time-tick 到来再 check
```

**关键性质**：
- **τ = 0 → strong consistency**：query 必须等到 subscriber 看到所有 ≤ L_r 的写入
- **τ = ∞ → eventual consistency**：query 立即用当前可见数据，无等待
- **0 < τ < ∞ → bounded staleness**：query 看到的数据**不会过期超过 τ virtual time**

## Time-tick 机制（实现支撑）

[guo-2022-manu §3.4]

为让 subscriber 知道"数据更新进度"，Manu 在 log channel 周期插入特殊控制消息 **time-tick**：

```
TSO（Time Service Oracle）发布 hybrid logical clock 时间戳：
   timestamp = (physical_time, logical_seq)
   - physical_time：与 wall clock 对齐
   - logical_seq：同物理秒内 event 顺序

Time-tick 机制：
   - 每 WAL channel 周期插入 time-tick 消息（类 Apache Flink watermark）
   - Subscriber 每收到 time-tick T 即知"≤T 的所有事件已收到"
   - τ 用 virtual time 表示，**实际就是 physical time**（因为 hybrid clock 第一分量与 wall clock 对齐）

→ user 可写："看 ≤10s 之前的数据"，Manu 直接转化为 timestamp 比较
```

## 与 wiki 其他一致性模型的对比

| 模型 | 形式 | 特例 | 工业代表 |
|---|---|---|---|
| Strong | 每读必最新 | δ=0 of delta | 传统 RDBMS、ZooKeeper |
| **Delta**（NEW，本概念） | bounded staleness τ | strong=0 / eventual=∞ | **Manu / Milvus 2.x** |
| Eventual | 最终一致 | δ=∞ of delta | DynamoDB、Cassandra |
| Read-your-writes | session 内见自己写 | 弱 delta 变体 | session-store |
| Snapshot isolation | query 看一致 snapshot | 与 delta 正交 | [Milvus](../systems/milvus.md) 1.x |

> **wiki 解读**：snapshot isolation（读看一致 snapshot）与 delta consistency（读看 ≤τ 之前的数据）**正交**——前者关心"是否一致"，后者关心"过期多久"。Manu 同时实现两者：snapshot 在 segment 级，delta 通过 time-tick + LSN 控制。

## 与 wiki 现有 source 的关系

- [systems/milvus.md] v2.6.x 文档没**显式**形式化 delta consistency，但 Streaming Node + WAL + segment-level snapshot 都是该模型的实现
- [concepts/woodpecker.md] Woodpecker 是 Manu time-tick + log 思路在 2.6 的演化（zero-disk WAL 替代外部 broker）
- [systems/spfresh.md] SPFresh version map + tombstone 与 delta consistency 不同——SPFresh 关心"vector 版本"，delta 关心"时间过期"

## Open Questions

- **跨 region delta consistency**：跨 cloud region 的 hybrid clock 同步精度？τ 在多 region 下的实际 staleness 上限？docs / paper 都未深入
- **Delta 与 GC 的交互**：tombstone GC 时机是否考虑 max τ in-flight queries？paper 未明示
- **Per-query τ**：用户能否每次 query 不同 τ？paper 暗示可以但未给 API
- **Delta 在 Pinecone / DiskANN / SPANN 等系统的引入**：理论上其他系统可移植 delta consistency，但 wiki 内仅 Manu 实证
- **Token-level vs vector-level delta**：multi-vector / late-interaction 场景下不同字段的 staleness 是否独立？paper 未涉及

Cited by: 待 query 引用
