---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/vespa.md, topics/disk-vs-memory-ann.md]
cited-count: 4
---

# Post-snapshot (kusupati-2022-matryoshka): scale-tier-shifts

## TL;DR (delta from radford-2021-clip post)

**MRL 在 tier-shift 表上**不改变 tier 数, 但**显著推迟 tier 切换**——每 tier 内 storage 可降 4× (1024-d → 256-d prefix), 让 same scale 更 cheap 更 deferred 切 disk tier. **关键 NEW**: tier 4 (100B-1T) 之前必须 disk tier (SPANN/DistributedANN); MRL prefix 256-d 让 100B × 256-d × int8 = 25.6 TB → 16 节点 × 1.6 TB SSD 可承——**100B 规模重新进入 single-stage memory + SSD 可行域**, 不必走 distributed graph (DistributedANN).

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| Tier-shift criteria | scale × storage × parallel | **+ MRL: 单 corpus 多 dim tier 内置** |
| Tier 切换 trigger | 单 dim storage 撞墙 | **MRL prefix 让 dim 是 application-time decision** |
| Tier 5 (≥1T) production case | DistributedANN + Turbopuffer | **不变, 但 MRL prefix 让 1T × 256-d 也可行** |

### MRL-aware tier-shift 表（updated 2026-05-11 post kusupati-2022-matryoshka）

| Tier | Pre-MRL storage | + MRL prefix (256-d truncate) | + MRL + int8 (post-hoc) |
|---|---|---|---|
| ≤ 1B (1024-d) | 4 TB float32 | 1 TB | 256 GB |
| 1-10B | 40 TB float32 | 10 TB | 2.5 TB |
| 10-100B | 400 TB float32 | 100 TB | 25 TB |
| 100B-1T | 4 PB float32 | **1 PB** | **256 TB** |
| ≥ 1T | 40 PB+ float32 | 10 PB+ | 2.5 PB+ |

→ **MRL + int8 让 1T scale 进入 SSD-feasible 范围** (16 节点 × ~20 TB SSD/node = 320 TB total).

### MRL 在 each tier 内的 path 拓宽

[per kusupati-2022-matryoshka + wiki tier-shift]

**Tier 1 (≤1B)**:
- Pre-MRL: HNSW + memory all-in (full 1024-d, 4 TB)
- + MRL: HNSW + 256-d prefix in memory (1 TB), 1024-d full optional on SSD for rerank
- **Tier 1 内 MRL 让单节点 vector DB capacity 提升 4×**

**Tier 2 (1-10B)**:
- Pre-MRL: HNSW + quantization 来 fit memory
- + MRL: HNSW + 256-d prefix (memory) + optional 1024-d full (SSD rerank); 不必 binary/PQ
- **MRL replaces post-hoc quantization 在 Tier 2 中**

**Tier 3-4 (10B-1T)**:
- Pre-MRL: 必须 SPANN/DiskANN/DistributedANN
- + MRL: **256-d prefix in memory + 1024-d full on SSD for rerank** — DiskANN-style 但 dim 调整
- **MRL + DiskANN-style** 是 Tier 3-4 内**最 cost-effective 组合**

**Tier 5 (≥1T)**:
- DistributedANN-style 仍主流, 但 MRL prefix 让 head index size 减小 (e.g., 2.5B head index × 256-d 而非 2048-d = 8× memory saving)
- Bing DistributedANN paper 不明示 MRL adoption

### Tier-shift 决策驱动（updated 2026-05-11 post kusupati-2022-matryoshka）

1. **Tier 1 (≤1B)** + MRL: HNSW + 256-d prefix (cheaper than full)
2. **Tier 2 (1-10B)** + MRL: HNSW + 256-d prefix, **跳过 post-hoc quantization**
3. **Tier 3 (10-100B)** + MRL: prefix in memory + full on SSD = **MRL-style adaptive DiskANN**
4. **Tier 4 (100B-1T)** + MRL: 不必走 DistributedANN, single-stage MRL + SSD 可承
5. **Tier 5 (≥1T)** + MRL: DistributedANN-style + MRL head index 8× memory saving

### 已知盲区

- **MRL × DistributedANN production combo**: paper 不明示
- **MRL + DiskANN-style adaptive (prefix in mem + full on SSD)**: production 实测 zero coverage
- **Tier 切换 threshold 在 MRL 之后**: 完整数据点不存在公开
- **DBMS 端 MRL prefix dim 决策**: query-time dynamic prefix selection 接口 ergonomics 不公开

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/vespa.md](../../systems/vespa.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
