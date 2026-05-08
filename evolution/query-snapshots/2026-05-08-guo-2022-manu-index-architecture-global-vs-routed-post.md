---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [queries/index-architecture-global-vs-routed.md, systems/milvus.md, concepts/delta-consistency.md, concepts/manu-ssd-hierarchical-kmeans.md, topics/in-place-vs-out-of-place-updates.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (guo-2022-manu): index-architecture-global-vs-routed

## TL;DR (delta from pinecone-docs post)

**Manu 给 (c) 层次路由的学术形式化** + 实测数据。"Manu = Milvus 2.x 学术论文" + 已有的 milvus-docs 共同构成 (c) 路由 DBMS 形态最完整覆盖。**Manu §3-4 的 schema → collection → shard → segment 路由层级**与 Pinecone serverless 的 namespace → slab 同代但不同形式——两者都是 (c) 路由的 cloud-native 形态。

## Answer

### Manu (c) 层次路由形态学术形式化（NEW）

[per systems/milvus.md "Manu academic basis"]

Manu §3 给 (c) 路由的形式化层级：

```
Schema（数据 schema）
  └─ Collection（= 关系 DB 的 table）
       └─ Shard（按 primary key hash，路由 channel）
            └─ Segment（512MB default，data placement 单位）
                 ├─ Growing segment（in-memory + WAL，可写）
                 └─ Sealed segment（immutable，可索引）
```

→ Manu 的 (c) 路由比 Milvus docs 描述更**学术形式化**：明确 schema → collection → shard → segment 四级；明确 growing vs sealed segment 的 state machine；明确 LSM merge / segment 的 512MB threshold 等。

### Manu vs Pinecone 的 (c) 路由对比（NEW）

[per systems/milvus.md + concepts/pinecone-serverless-slabs.md]

| | **Manu (Milvus 2.x)** | **Pinecone Serverless** |
|---|---|---|
| 路由层级 | schema/collection/shard/segment（四级） | project/index/namespace/slab（四级） |
| Multi-tenant 单元 | partition_key | namespace |
| Segment / slab 大小 | 512 MB default（Milvus 2.x） | 不公开 |
| 状态 | growing → sealed → merged | memtable → flush → slab → merge |
| WAL / 时间一致性 | TSO + time-tick + LSN | LSN |
| Consistency 模型 | **Delta consistency τ**（user-tunable） | best-effort（on-demand）/ 强 cache（DRN） |
| 索引算法 | 用户选 segment-level index_type | **Adaptive across lifecycle** |
| Index 演化 | segment 内固定（生命周期内不变） | **Adaptive merge upgrades 算法** |
| Cloud-native | ✓（K8s + S3） | ✓（云原生 SaaS） |

→ Manu 与 Pinecone 是**同代不同 idiom**——Manu 偏开源 DBMS（用户控制 segment 级索引），Pinecone 偏 SaaS 黑盒（用户不控制）。两者**架构骨架同源**（cloud-native + log backbone + segment/slab merge）。

### Delta Consistency 让 (c) 路由更灵活（NEW）

[per concepts/delta-consistency.md]

之前 (c) 路由的"刚性"在 update 与 query 协调上：
- Read-after-write：query 必须看到自己刚写的（强 consistency）
- Stale-OK reads：query 可以看到任意旧 view（eventual）

Manu 的 delta τ 让 (c) 路由能**精细控制 query 命中哪些 segment**：
```
Per-query τ 决定：
   query 是否等 latest segment merge?
   query 是否包含未 flush 的 memtable 数据?
   query 是否容忍 ≤τ 时间前的删除标记?
```

→ (c) 路由 + delta consistency 是 vector DBMS 的精细化 query 语义。

### 与之前 ingest 的演进

| | pinecone-docs post | **guo-2022-manu post (NEW)** |
|---|---|---|
| (c) 系统数 | 4（Meta/SPANN/Milvus 1.x/Milvus 2.x/SPFresh/Pinecone） | 不变（Manu = Milvus 2.x） |
| (c) 形式化 | docs 级 + Pinecone llms-full.txt | **+ Manu VLDB 论文学术形式化** |
| Consistency 模型 | 各系统隐式 | **+ Manu delta consistency 显式** |
| Cloud-native 设计文献覆盖 | docs only | **+ 学术论文** |

### 已知盲区

- **千亿/万亿规模 (c) 实测**：Manu §5 实测仅到 100M
- **Pinecone vs Manu 路由 overhead 对比**：双方都不公开
- **跨 region (c) 路由延迟**：Manu 论文未涉及
- **Manu DRN-style provisioned read 模式**：Manu 没有 Pinecone DRN 这种"专用 read hardware" 概念

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/delta-consistency.md](../../concepts/delta-consistency.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
