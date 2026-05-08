---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/analyticdb-v.md]
cited-count: 3
---

# Post-snapshot (wei-2020-analyticdb-v): hnsw-vs-nsg-selection

## TL;DR (delta from guo-2022-manu post)

**ADBV 显式选 HNSW 作 streaming layer 唯一 graph 选项**——不在 batching layer 用 graph。论文 §3.2 给具体理由：HNSW 支持 real-time insert；但**内存重 + 不适合 offline batch 大数据**——所以 batching 走 quantization 路径（[VGPQ](../../concepts/vgpq.md)）。这与 [Milvus/Manu](../../systems/milvus.md) 在所有 segment 类型都让用户选 graph 不同——ADBV **强约束 streaming = HNSW**。

## Answer

### ADBV 的 graph 选择理由（NEW）

[per systems/analyticdb-v.md "Lambda 三层框架"]

```
Streaming layer:
   - HNSW（in-memory，real-time insert）
   - 不用 NSG（NSG 不支持增量插入）
   - 不用 VGPQ（VGPQ 需 offline 训练 codebook）
Batching layer:
   - VGPQ（offline 训练 + 大数据 + 压缩）
   - 不用 HNSW（baseline 数据量大 → 内存爆）
```

→ ADBV 的设计强制 graph = HNSW；论文论证清晰：
- **HNSW 的优势**（vs NSG）：增量 insert 支持是 streaming layer 必要条件
- **HNSW 的劣势**（内存重）在 batching layer 通过用 quantization 路径绕开

→ 这是**首次** wiki 内见 system 显式拒绝 NSG 的具体设计 reason——不是 NSG 算法本身次于 HNSW，而是 NSG 的"不增量"特性不匹配 streaming 角色。

### 为什么 ADBV 不在 batching layer 用 graph？

[per systems/analyticdb-v.md, wei-2020-analyticdb-v §3.2]

论文 §3.2 给数字：
- HNSW 每 record ~400 byte 内存（仅 index，不含原向量）
- baseline 数据量大（PB 级）→ 全 HNSW 内存爆
- 所以走 quantization 路径（VGPQ 压缩 + 存 Pangu 分布式存储）

→ **HNSW 在大规模下的内存瓶颈**驱使 ADBV 走"graph 仅在小数据 + quantization 在大数据"两阶段路线。这与 [Faiss](../../systems/faiss.md) factory string `IVF65536_HNSW32,PQ32`（HNSW-as-coarse-quantizer）思路类似——**graph 仅做 routing，不做 final search**。

### 决策树更新（updated）

[per topics/index-selection.md]

| 场景 | 推荐 | 备注 |
|---|---|---|
| Streaming / 实时 insert + 小数据 | **HNSW** | NSG 不支持增量；ADBV 实证 |
| Batching / 大数据 + offline | **Quantization (PQ/VGPQ/SCANN)** 或 graph-as-coarse-quantizer | ADBV / Faiss 同思路 |
| 全数据全 graph | 不推荐 > 100M | 内存爆；除非 SSD 路线（DiskANN/SPANN） |

### 与之前 ingest 的演进

| | 前几轮 post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| 选 HNSW vs NSG 的理由 | 工程比较（生态、性能） | **+ 系统强约束**：streaming 必用 HNSW（NSG 不增量） |
| Graph + quantization 组合 | Faiss `IVF_HNSW,PQ` 提及 | **+ ADBV streaming HNSW + batching VGPQ 双层 graph 不全用** |
| Graph 在大数据下的内存瓶颈 | 隐含 | **显式数字 400 byte/record** |

### Open / 未覆盖

- **NSG 在 streaming layer 的真实禁区**：NSG 论文 §5 明示"不支持增量"——ADBV 直接绕开；但若有 incremental NSG 变体（如某种 in-place NSG）是否能解？wiki zero coverage
- **ADBV 在小批量 update 下 HNSW 内存增长**：streaming layer 内存阈值触发 merge——具体阈值与 latency 影响 docs 未深入

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
