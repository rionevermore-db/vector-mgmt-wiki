---
title: FreshDiskANN（首个 graph-based billion-scale streaming ANN 系统）
type: system
sources: [singh-2021-freshdiskann]
related: [diskann.md, spann.md, spfresh.md, milvus.md, faiss.md, starling.md, ../concepts/vamana.md, ../concepts/freshvamana.md, ../concepts/lire.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../topics/disk-vs-memory-ann.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/index-selection.md, ../benchmarks/freshdiskann-streaming-sift800m.md]
created: 2026-05-09
updated: 2026-05-09
---

# FreshDiskANN

**TL;DR**: Microsoft Research India 在 arXiv 2021 提出的 **billion-scale streaming graph ANN 系统**——把 [DiskANN](./diskann.md) 从 static 升级为支持实时 insert/delete。核心架构是 **Long-Term Index (LTI) on SSD + Temp Index (TempIndex) in DRAM + StreamingMerge background process**：(a) [FreshVamana](../concepts/freshvamana.md) 是单 index 的 graph 算法（α-RNG property 维持 navigability under streaming），(b) StreamingMerge 是 two-pass SSD-write-efficient 合并过程。**实测 800M SIFT 索引：1800 inserts/sec + 1800 deletes/sec sustained, burst 40K inserts/sec, search 5-15ms 同时 95+% recall, single-machine 128GB DRAM**——把 PLSH（state-of-art LSH-based fresh-ANNS）的 25 台机器需求压到 **1 台**（**5-10× 成本降**）。是 [SPFresh](./spfresh.md) cluster-path 的"姐妹工作"（同 Microsoft Research，FreshDiskANN 早 2 年），也是 [Manu](./milvus.md) §3.5 stream indexing 的 graph-path 完整版。

## 与 wiki 现有系统的定位差异

[per topics/in-place-vs-out-of-place-updates.md, singh-2021-freshdiskann §1, §2]

**关键 Insight**：之前 wiki 内 streaming-friendly 系统全部 cluster-based（SPFresh / Manu / ADBV）；graph-based path 一直 flagged 为"开放问题"。FreshDiskANN 闭合这个 gap。

| | [DiskANN](./diskann.md) | [SPFresh](./spfresh.md) | [Manu](./milvus.md) stream | **FreshDiskANN** |
|---|---|---|---|---|
| 索引算法 base | Vamana | SPANN (cluster) | IVF-FLAT temp | **FreshVamana (graph)** |
| Streaming insert | ✗（static） | ✓ via LIRE | ✓ via temp index | **✓ via FreshVamana** |
| Streaming delete | ✗ | ✓ via LIRE 5 操作 | n/a (segment level) | **✓ via lazy delete + batch consolidate** |
| Real-time freshness | ✗（周期 rebuild） | ✓ | ✓ | **✓ <1ms latency for insert/delete** |
| 1B 实证 | static 1B SIFT | 1B SIFT | 100M scale | **1B SIFT (800M steady-state demo)** |
| 单机 budget | 64 GB RAM + SSD | 1B 4GB RAM + raw SSD via SPDK | n/a | **<128 GB RAM** for 1B |
| Update cost vs full rebuild | n/a (rebuild only) | DiskANN rebuild 32 cores × 2 days | n/a | **5.25× faster** (15832s vs 83140s on 800M) |
| 论文年份 | 2019 NeurIPS | 2023 SOSP | 2022 VLDB | **2021 arXiv (preprint)** |
| 团队 | Microsoft Research（Subramanya 等）| Microsoft Research（Xu 等）| Zilliz | **Microsoft Research India + CMU（Singh, Subramanya 等）** |
| 与 DiskANN 同团队 | n/a | 不同 | 不同 | **是（Subramanya/Krishnaswamy/Simhadri 共作）** |

**核心论点**：FreshDiskANN 不是新算法概念，而是"DiskANN + 工程让 streaming 可行"。它的位置是 **DiskANN 的 streaming-ready 版本**——这是 graph path 在 fresh-ANNS 维度的工业完整答案。

## 架构图

[singh-2021-freshdiskann §5.1]

```
┌─────────────────────────────────────────────────────┐
│  RAM (~128 GB peak for 1B index)                    │
├─────────────────────────────────────────────────────┤
│  RW-TempIndex（FreshVamana, accept inserts）        │
│    ↓ periodic snapshot (RW → RO)                   │
│  RO-TempIndex(es)（read-only 多实例, ≤6 typical）   │
│    ↓ background StreamingMerge                      │
│  DeleteList（被删点 ID）                            │
│  Δ data structure（merge 期间 backward edges buffer）│
│  PQ short codes（25-32 bytes/point for LTI）        │
│  Crash redo-log buffer                              │
├─────────────────────────────────────────────────────┤
│  SSD (NVMe ~3.2 TB)                                 │
├─────────────────────────────────────────────────────┤
│  Long-Term Index (LTI)                              │
│    - SSD-resident FreshVamana graph                 │
│    - 全精度 vector + neighbor list                  │
│    - 与 DiskANN 同 layout (compatible)              │
│  Crash redo-log（持久化 RW-TempIndex 操作）         │
└─────────────────────────────────────────────────────┘
```

## 数据流 / 控制流

### Insert 路径（fast path）

[singh-2021-freshdiskann §5.2]

```
client.Insert(x_p):
  1. Acquire RW-TempIndex lock
  2. Run FreshVamana Algorithm 2:
     - GreedySearch on RW-TempIndex graph
     - RobustPrune to set N_out(p)
     - Bi-directional edge update
  3. Append to redo-log
  4. Return → user latency ~1 ms

Background:
  When RW-TempIndex grows to ~5M points:
    snapshot RW-TempIndex → new RO-TempIndex instance
    create empty new RW-TempIndex
```

### Delete 路径（O(1) fast path）

```
client.Delete(p):
  1. Append p to DeleteList
  2. Append to redo-log
  3. Return → user latency ~0.1 μs
```

### Search 路径

```
client.Search(x_q, K, L):
  Parallel:
    - Search LTI on SSD (DiskANN-style)
    - Search RW-TempIndex (FreshVamana in DRAM)
    - Search each RO-TempIndex (FreshVamana in DRAM)
  Merge results from all sources
  Filter out points in DeleteList
  Return top-K
```

**Search latency**：LTI search 用 DiskANN 同样 logic（PQ navigation + SSD full-precision rerank）；TempIndex search 是 in-memory FreshVamana（极快）。Mean latency 5-15 ms steady state；merge 进行时偶尔 spike。

### StreamingMerge 过程（核心）

[singh-2021-freshdiskann §5.3]

**触发**：当 cumulative TempIndex 内存接近预设上限（例：30M points / 13 GB），后台启动 merge：

```
Inputs:
  P = current LTI on SSD（800M points typical）
  N = points in (RO-TempIndex)s（30M typical）
  D = DeleteList（30M typical）

Output:
  New LTI = (P ∪ N) \ D  on SSD

Phase 1 - Delete Phase:
  Sequential pass over P on SSD（block-by-block）
  For each block, multi-thread:
    For each point p in block s.t. N_out(p) ∩ D ≠ ∅:
      Run FreshVamana Algorithm 4 (RobustPrune with α)
      Use PQ approximate distance for distance comparisons
  Write modified block back → intermediate-LTI on SSD

Phase 2 - Insert Phase:
  For each new point p in N:
    GreedySearch(s, x_p, 1, L) on intermediate-LTI（SSD reads）
    RobustPrune to set N_out(p)
    !!! 不立即写 backward edges（避免 R IOPS per insert）
    Buffer (p, N_out(p)) backward edges to in-memory Δ data structure
  Δ size: O(|N|·R)，~7 GB for |N|=30M, R=64

Phase 3 - Patch Phase:
  Sequential pass over intermediate-LTI（block-by-block）
  For each point p in block:
    Check if Δ(p) contains backward edges to add to N_out(p)
    If |N_out(p) ∪ Δ(p)| > R: RobustPrune
    Else: append Δ(p) to N_out(p)
  Write final block to SSD → new LTI

Cleanup:
  Drop processed RO-TempIndex instances
  Done
```

**关键复杂度**：
- I/O: **2 sequential SSD passes** over LTI（Delete + Patch）
- Memory: **O(|D| + |N|·R)**——与 change size 成正比，**与 LTI 总大小无关**
- Compute: ~O(|N|·L) for inserts + O(|D|·R²) for deletes

**实测**（800M SIFT, 30M change, 40 threads）：
- StreamingMerge: **15832 s ≈ 4.4 h**
- Equivalent DiskANN full rebuild: **83140 s ≈ 23 h**
- → **5.25× faster**

→ FreshDiskANN 把 update cost 从"O(全 index size)"降到"O(change size)"——是 fresh-ANNS 工程上的根本性改进。

## 关键设计决策

### 1. 三层 index 结构（LTI + RO-TempIndex + RW-TempIndex）（§5.1）

不是简单"在 RAM 缓冲，定期 flush"——而是**精确分离 read-only 与 read-write**：

| | RW-TempIndex | RO-TempIndex | LTI |
|---|---|---|---|
| 数量 | exactly 1 | 0+ instances | exactly 1 |
| 存储 | RAM | RAM | SSD |
| 接受 insert | ✓ | ✗ | ✗（仅通过 merge） |
| 接受 search | ✓ | ✓ | ✓ |
| 数据格式 | full FreshVamana graph + 全精度 vector | 同 RW-TempIndex（snapshot） | DiskANN-style + PQ short codes |
| Search performance | 极快（DRAM） | 极快 | 较慢（SSD） |

**Trade-off**：复杂度高 vs (a) inserts 不阻塞 search，(b) crash recovery 仅需重放 RW-TempIndex redo-log（小），(c) merge 不需要锁全 index。

### 2. PQ-based approximate distance during merge（§5.3）

merge 时**不读 LTI 的全精度 vector**——直接用 in-memory PQ short codes 算 approximate distance：
- DiskANN PQ codes 已经在 RAM（25-32 bytes/point for 1B = 32 GB）
- merge 时距离计算用 PQ → 节省 99% SSD reads
- **代价**：merge 后 graph 是基于 approximate distance 构造——recall 略降

实测（§5.5 Fig 4）：800M SIFT 40 cycles 后 5-recall@5 从 ~95% 降到 ~92.5%——**3% 降但仍稳定**（不再下降）。Trade-off acceptable。

### 3. Lazy delete + batch consolidation（§5.2 + §4.2）

详见 [concepts/freshvamana.md "Delete 算法"](../concepts/freshvamana.md)。User-perceived delete latency O(1)；真正 graph 修改在 StreamingMerge 时统一做。

### 4. Δ in-memory data structure 暂存 backward edges（§5.3）

**问题**：Insert 阶段需要 backward edges (p_in, p)；如果 eagerly 写 SSD → R 个随机 IOPS per insert。

**解决**：buffer 到 in-memory Δ；Phase 3 (Patch) 用 block-sequential 方式批量 patch。Δ 大小 ~|N|·R = 30M × 64 ≈ 1.92 GB pointers——可控。

### 5. Crash recovery via redo-log（§5.6）

RO-TempIndex + LTI 都 read-only → 自动 persistent。仅 RW-TempIndex 是 mutable in-RAM —— 用 redo-log 记录每个 insert/delete 操作。Crash 后 replay log。

**Snapshot 频率 trade-off**：snapshot 太频繁 → 多 RO-TempIndex 实例 → search 时多搜几个 → 慢；太少 → recovery 时间长。论文 default 5M points per RW-TempIndex，~6 RO-TempIndex max。

## Scale 边界

[singh-2021-freshdiskann §6]

| 配置 | 数据 | 实测 |
|---|---|---|
| **mem-mc**: Azure E64d_v4 (64-vcore VM) | SIFT100M / Deep1M / GIST1M | FreshVamana RAM 实测 |
| **ssd-mc**: 2× Xeon 8160 (48 cores, 96 threads) + 3.2 TB Samsung PM1725a NVMe | **SIFT800M steady-state** | **FreshDiskANN 完整系统**——1800+1800 inserts/deletes/sec, 1000 search/sec, 95+% recall, ~125 GB peak RAM |
| 同硬件 ramp-up 阶段 | SIFT 100M → 800M via 3500 inserts/sec | 3 days, ~125 GB peak RAM throughout |
| 同硬件 burst 实测 | bursts | **40,000 inserts/sec** burst capacity |
| 论文未实测 | trillion-scale | **distributed deployment**（broadcast queries + route updates）—— logical extension |

> **wiki 解读**：800M ssd-mc 实测是单机最大；1B+ 需 distributed setup。这与 [DiskANN single 1B](./diskann.md) / [SPANN Bing 几千亿](./spann.md) / [SPFresh 1B](./spfresh.md) 的 single-machine 路径并列——graph-path 现在有完整工业 streaming 选项。

## 实测亮点

[详见 benchmarks/freshdiskann-streaming-sift800m.md](../benchmarks/freshdiskann-streaming-sift800m.md)

- **Steady state throughput**：800M index, 1800 ins/sec + 1800 del/sec + 1000 search/sec @ 95+% 5-recall@5
- **Insert latency**：mean ~1 ms (Fig 6 right)
- **Delete latency**：~0.1 μs (just append to DeleteList)
- **Search latency**：mean 5-15 ms steady; spike up to 40 ms during merge Patch phase
- **Burst capacity**：40,000 inserts/sec for short bursts
- **Update cost**：StreamingMerge 5.25× faster than full DiskANN rebuild
- **Recall stability**：80M / 800M index 40+ cycles, 92-95% recall stable
- **vs PLSH**：~25× fewer machines for same 1B fresh-ANNS task

## 与 wiki 已有系统的对比

### 与 [DiskANN](./diskann.md)（直接前身）

[singh-2021-freshdiskann §1-2]

FreshDiskANN 是 DiskANN 的 **streaming-ready 升级版**：

| | DiskANN | **FreshDiskANN** |
|---|---|---|
| 索引基础 | static Vamana | **FreshVamana**（α=1.2 streaming-stable） |
| 数据结构 | LTI only on SSD | **LTI on SSD + TempIndex in DRAM + DeleteList** |
| Insert/Delete | ✗ | **✓ <1 ms latency** |
| Update mechanism | full rebuild | **StreamingMerge**（5.25× faster） |
| 1B 实证 | 1B SIFT static | 800M SIFT streaming（论文止步） |
| Search latency | reference | comparable + 偶尔 spike during merge |
| Memory | 64 GB | ~125 GB peak（多 TempIndex 实例 + Δ） |
| Production | DiskANN open-source | **同 microsoft/DiskANN repo** |

→ **FreshDiskANN 应替代 DiskANN 作 default static disk graph**——但论文未明示是否所有 DiskANN deployment 都迁移。当前 microsoft/DiskANN GitHub 同仓库提供两套接口。

### 与 [SPFresh](./spfresh.md)（cluster-path 姐妹工作）

[每 systems/spfresh.md + concepts/lire.md vs 本 page]

| | SPFresh / LIRE | **FreshDiskANN / FreshVamana** |
|---|---|---|
| 论文年份 | SOSP 2023 | arXiv 2021（preprint） |
| 索引 base | SPANN (inverted file) | Vamana (graph) |
| 路径 | **cluster-based** | **graph-based** |
| 增量机制 | LIRE 5 操作（split/merge/reassign）+ 2 NPA 必要条件 | FreshVamana α-RNG + lazy delete consolidation |
| 内存预算 | ~4 GB for 1B | ~128 GB for 1B |
| Update cost | 极低（per-cluster locality）| 中（StreamingMerge over LTI 仍需 sequential SSD pass） |
| Recall stability 实证 | 100 days × 1% daily | 50 cycles × 5%/10%/50% |
| 1B 实证 | ✓ | ✓ (800M demo) |
| 团队 | Microsoft Research（Yuming Xu 等） | Microsoft Research India + CMU（Singh + Subramanya 等） |
| 共同点 | 都是 Microsoft Research，都是 fresh-ANNS 1B billion-scale 工业方案 | 同上 |

→ FreshDiskANN 早 2 年；SPFresh 在 cluster path 实现等价能力 + 更小内存预算。**两条路径并存 in production**：graph 适合"高 recall，可接受较大 RAM"；cluster 适合"极小 RAM 预算"。详见 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)。

### 与 [Manu](./milvus.md) §3.5 stream indexing

[guo-2022-manu §3.5]

Manu's stream indexing：per-segment growing data → 每 10K vector slice → temp IVF-FLAT → search 用 temp index 而非 brute-force。

| | Manu stream indexing | **FreshDiskANN** |
|---|---|---|
| 索引层 | per-segment IVF-FLAT temp | **per-segment FreshVamana** (理论上) |
| 长期索引 | sealed segment 后建完整 index | **LTI on SSD + StreamingMerge** |
| 内存 vs 性能 | IVF-FLAT 小但 search 慢 | FreshVamana 中等 RAM + 快 search |
| 工业实证 | Milvus 2.x production | Microsoft / DiskANN 用户 |

→ **Manu 的 stream indexing 是 segment-level 的概念雏形**；FreshDiskANN 是单 LTI 的完整 fresh-ANNS。两者**可结合**：Manu segment 用 FreshDiskANN 作 disk graph + Manu stream indexing 作 segment-internal growing buffer——理论上 Milvus production 可以采用这种 hybrid。

### 与 [Starling](./starling.md)（同 Zilliz/Milvus team disk graph）

[wang-2024-starling §7 Discussion]

Starling §7 明示参考 FreshDiskANN-style "static disk index + 动态 in-memory + 周期 merge" update model：

> "Some vector databases (e.g., ADBV [70]) segregate dynamic and static indexes to facilitate updates. The dynamic index, residing in memory, is built incrementally and uses a bitset to monitor deleted data. As incoming data continues to grow, the dynamic index expands correspondingly. Consequently, an asynchronous merging process transfers the burgeoning dynamic index to the disk-based static index, necessitating a comprehensive index reconstruction."

→ Starling 的 update model 是 **FreshDiskANN-derived**——但 Starling 论文重点是 segment-level disk layout（block shuffling），update 模型仅在 §7 Discussion 提及，未深入实证。**Starling + FreshDiskANN 完整集成是 logical work**。

### 与 [Faiss](./faiss.md)

Faiss 论文 [douze-2024-faiss-library §6.1] 把 FreshDiskANN [78] 列为"data update with periodic rebuild"的 emerging direction（**关闭** Faiss page 的 "FreshDiskANN is in development" Open Q）。Faiss 当前 release 不集成 FreshDiskANN——是 microsoft/DiskANN 同仓库的功能。

## 生产案例

- **GitHub**: [microsoft/DiskANN](https://github.com/Microsoft/DiskANN)——FreshDiskANN 与 DiskANN 共同仓库，C++17
- **Production deployment**：论文未明示具体公司部署；推断 Microsoft Bing Vector Search 等内部产品潜在用户
- **License**：MIT

## Open Questions

- **HNSW / NSG 的 streaming 等效改造**：FreshVamana 因 Vamana 已有 α 参数，自然引入 α=1.2；HNSW / NSG 没有等价参数——能否给 HNSW 加类似 α-RNG？理论上 yes，但工业实现未做。**这是 FreshDiskANN 等价 hnswlib / NSG-Disk 的 logical next work**——论文未涉及
- **Trillion-scale distributed deployment**：单机 800M 实证 OK；trillion-scale 需 broadcast queries + route updates——具体协议（query coordinator / consistent hashing / consensus）论文 §1 仅提及概念
- **FreshDiskANN + [filter-aware](../concepts/filtered-vamana.md)**：FreshVamana + FilteredVamana 联合理论可行未实证；filter set 演化下的 recall stability 完全未涉及
- **FreshDiskANN + [block shuffling](../concepts/block-shuffling.md)**：StreamingMerge 后 OR(G) 漂移；何时触发 block shuffle 重跑？周期性 vs 自适应？wiki 完全空白
- **FreshDiskANN + [RaBitQ](../concepts/rabitq.md) quantizer**：当前用 PQ short codes for routing；RaBitQ unbiased + sharp bound 替换后 merge 决策更准 → 减少 disk reads + 减少 graph error。两个论文相互不知，未实证
- **VBASE iterator + FreshDiskANN**：theoretically VBASE engine + FreshDiskANN as backing index 可叠加（FreshVamana 满足 RM）；论文相互不知，未实证
- **Concurrent insert/delete + serializable isolation**：fine-grained locking OK；strong consistency vs delta consistency 的 trade-off 论文未深入
- **Snapshot 频率自适应**：current 5M points per RW-TempIndex 是 fixed parameter；workload-aware adaptive 未做
- **embedding model 升级**：与 wiki 全 frontier 一致——StreamingMerge 不能跨 model；model 升级仍需 full rebuild
- **PLSH cost 25× 是论文数据**：未独立 verify；PLSH paper [54] 是 2013 年——到 2021 已有更新 LSH 系统，对比可能需更新

Cited by: 待 query 引用
