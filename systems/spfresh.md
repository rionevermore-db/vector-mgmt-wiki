---
title: SPFresh（Incremental In-Place Update System）
type: system
sources: [xu-2023-spfresh, chen-2021-spann, singh-2021-freshdiskann, turbopuffer-docs]
related: [spann.md, diskann.md, milvus.md, faiss.md, freshdiskann.md, turbopuffer.md, ../concepts/lire.md, ../concepts/freshvamana.md, ../concepts/product-quantization.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/spfresh-vs-diskann-spann-update.md, ../benchmarks/freshdiskann-streaming-sift800m.md]
created: 2026-05-07
updated: 2026-05-11 (Turbopuffer = first OSS-known production deployment — SPFresh frontier closed)
---

# SPFresh

**TL;DR**: Microsoft Research Asia + USTC 开源的**首个支持 billion-scale 向量索引 in-place 更新**的系统。建在 [SPANN](./spann.md) 之上，加 [LIRE](../concepts/lire.md) 协议（增量再平衡）+ 自研 SPDK-based raw SSD 存储引擎。100 days × 1% daily update 模拟下 P99.9 latency 稳定 ~4ms（DiskANN 在 rebuild 时飙到 >20ms）；rebuild 资源 1100 GB→**10 GB**、32 cores→**2 cores**——节省两个数量级。论文核心论点："single-node performance builds a strong foundation for the future distributed version"，**仍是单机系统**。[xu-2023-spfresh §1, §6]

## 与现有 wiki 系统的定位差异

| | [Faiss](./faiss.md) | [DiskANN](./diskann.md) | [SPANN](./spann.md) | [Milvus](./milvus.md) | **SPFresh** |
|---|---|---|---|---|---|
| 类型 | Library | 算法系统 | 算法系统 | DBMS | **算法系统 + 增量更新** |
| 存储 | DRAM/HBM | DRAM (PQ) + SSD | DRAM (centroids) + SSD | DRAM + S3 | **DRAM (centroids) + SSD raw** |
| 动态数据 | ✗ | ✗ | ✗ | ✓ (LSM) | **✓ (in-place + LIRE)** |
| 全局 rebuild 需求 | 低（library） | **必须**（streamingMerge） | **必须**（partition 累积 skew） | 取 segment 内 index | **不需要** |
| Rebuild 资源（1B SIFT） | n/a | 1100 GB + 32 cores × 2d | 260 GB + 45 cores × 4d | n/a | **持续 10 GB + 2 cores** |
| 数据漂移适应 | codebook freeze | rebuild 后才修复 | partition skew 累积 | LSM merge | **NPA 主动维持** |
| 形式化更新协议 | ✗ | ✗ | ✗ | ✗ | **✓ ([LIRE](../concepts/lire.md))** |

**核心论点**：SPFresh 把 [SPANN](./spann.md) 的 SSD-resident IVF 路线**从"静态 + 周期 rebuild"升级为"动态 + 增量 in-place"**。这是 SSD-resident ANN 第一次原生支持 update。

## 架构图

```
┌──────────────────────────────────────────────┐
│  Memory                                      │
│  ┌──────────────────────────────────────┐    │
│  │ SPTAG Index（centroids，沿用 SPANN）  │    │
│  └──────────────────────────────────────┘    │
│  ┌─────────────────┐   ┌──────────────────┐  │
│  │ Foreground:     │   │ Background:      │  │
│  │  Updater        │   │  Local Rebuilder │  │
│  │ ┌─────────────┐ │   │ ┌──────────────┐ │  │
│  │ │ VID Map     │ │   │ │ Job Queue    │ │  │
│  │ │ Version Map │ │   │ │ (split/      │ │  │
│  │ │ (7b ver +   │ │   │ │  merge/      │ │  │
│  │ │  1b tomb)   │ │   │ │  reassign)   │ │  │
│  │ └─────────────┘ │   │ │              │ │  │
│  │       Searcher  │   │ │ Threads      │ │  │
│  └─────────────────┘   │ └──────────────┘ │  │
│         │ feed-forward │   │              │  │
│         └──────────────►   │              │  │
│                            └──────────────┘  │
├──────────────────────────────────────────────┤
│  Block Controller                            │
│  ├─ Block Mapping (40 byte/posting)          │
│  ├─ Free Block Pool                          │
│  └─ Concurrent I/O Queue (SPDK ring)         │
├──────────────────────────────────────────────┤
│  Storage（NVMe SSD raw block via SPDK）      │
└──────────────────────────────────────────────┘
```

[xu-2023-spfresh Fig 5, 6]

## 数据流 / 控制流

### 写入路径（Updater 前台）

[xu-2023-spfresh §4.1]

1. 用户 insert v → SPTAG 找最近 centroid → APPEND v 到对应 posting tail
2. Updater 检查 posting 长度 → 超 split limit 则发 split job 给 Local Rebuilder
3. 用户 delete v → 在 version map 标 tombstone（7-bit version 增 + 1-bit tombstone）
4. **前台不做实际 SSD delete**——延迟到 reassign 或 GC 时批量清理

### 后台处理（Local Rebuilder）

- **Split job**：从 Updater 来；GC 已删除向量；若仍超 split limit → balanced clustering [SPANN §3.1]
- **Merge job**：从 Searcher 来（搜索时发现 posting 太短）；删除短 posting，append 到最近 posting
- **Reassign job**：split / merge 触发；按 [LIRE 2 必要条件](../concepts/lire.md) 检查附近 ≤64 posting 的 NPA 违反者

**Concurrency control**：fine-grained posting-level write lock 防止 split/merge/reassign 同时改同 posting。论文实测 lock contention <1%（通常仅小部分 posting 并发被改）。

### 读取路径（Searcher）

沿用 [SPANN](./spann.md) 路径：
1. SPTAG 内存索引找最近 K centroids
2. Query-aware dynamic pruning 缩到 ε₂ 内的候选
3. ParallelGET 批量读 posting blocks（SPDK 异步 I/O queue）
4. 用 version map 跳过 tombstone + 旧 version replicas

## 关键设计决策

### 1. 在 [SPANN](./spann.md) 之上而非新系统（§3, §4）

SPFresh 不重新设计 IVF 索引——直接复用 SPANN 的 SPTAG 内存索引 + 平衡分区算法 + posting list on disk 模型。**新增的只有 update 路径**。

**Trade-off**：保持 SPANN 的 Bing 千亿规模生产实测优势 vs 受限于 cluster-based 路线（不能扩展到 graph-based 索引）。

### 2. Foreground / Background pipeline（§4）

[LIRE](../concepts/lire.md) 操作切分到 Updater（写 path 关键路径）+ Local Rebuilder（背景 split/merge/reassign）。

**Trade-off**：bypass legacy storage stack vs SPDK 自定义实现复杂；前台延迟极低 vs 后台资源占用持续。

### 3. Raw SSD via SPDK（§4.3）

绕过 OS 文件系统 + KV store（Log-Structured Merge tree, RocksDB），直接 SPDK raw block：
- **Append-only 优化**：posting 是 immutable + 末尾 append；无 read-modify-write 上 KV 抽象的浪费
- **Block Mapping** 在内存（dense array, 40 byte/posting，1B 向量 ≈ 4 GB）
- **Free Block Pool** 管理 split/reassign 后释放的块
- 读延迟 = 单次 SSD I/O；写延迟 = 单次 append + atomic CAS 更新 mapping

**Trade-off**：极致 IOPS（饱和 NVMe 400K guarantee）vs 失去 OS layer 的故障恢复 / 备份生态——必须自实现 snapshot + WAL 恢复。

### 4. Version map 解决并发删除（§4.1, §4.2.2）

LIRE reassign 把向量 append 到新 posting + 删除旧 posting；并发 search 可能撞到中间状态。
- 每向量 1 byte version map（in-memory + on-disk）
- Reassign 时 atomic CAS（compare-and-swap）更新 version
- Search 跳过 stale version

**Trade-off**：每向量 1 byte 元数据 vs 真正 lock-free reassign。

### 5. Crash Recovery：snapshot + WAL（§4.4）

- 周期 snapshot：内存 SPTAG + version map + Block Mapping + Block Pool 全 flush 到 disk（1B 向量约 40 GB，PCIe NVMe 上 2-3 秒）
- 增量 WAL 记录两次 snapshot 之间的 update
- Block-level copy-on-write：snapshot 期间释放的块进 pre-release buffer，等下次 snapshot 后才进 Free Block Pool

**Trade-off**：snapshot 大但 PCIe NVMe 上够快 vs 必须延迟释放块（消耗暂时存储）。

## Scale 边界

[xu-2023-spfresh §5.3]

| 配置 | 数据 | 表现 |
|---|---|---|
| Azure lsv3 VM（16 vCPU, 128 GB RAM, NVMe local） | SIFT1B / SPACEV1B | search 4K QPS + update 2K QPS @ 15 cores 饱和 NVMe 400K IOPS |
| 同硬件 100 days × 1% daily update | SPACEV 100M | P99.9 ~4ms 稳定，10 GB memory + 2 cores at peak |
| 单 NVMe SSD bottleneck | — | IOPS 上限即系统上限（论文承认） |
| 论文未测 | 多 NVMe / 分布式 | §6 明示 "future distributed version" |

> **wiki 解读**：SPFresh 的极限是单 NVMe SSD 的 IOPS（typical commodity NVMe 400-1000K guarantee）。这意味着千亿规模 throughput 必须横向扩展——这是 [Milvus](./milvus.md) v2.6.x 的领域，而 SPFresh 论文未涉及。

## 与现有 wiki 系统的对比

### 与 [SPANN](./spann.md) 的关系

SPFresh 严格是 SPANN 的 superset：
- 全部 SPANN 设计（SPTAG + balanced clustering + posting list on SSD + closure clustering + query-aware pruning）保留
- 加 Updater + Local Rebuilder + Block Controller 三组件
- LIRE 协议保证 update 时 NPA 仍维持

**SPANN+ baseline**（论文 §5）= SPANN + append-only update **without** split/reassign。SPFresh = SPANN+ + LIRE。SPANN+ vs SPFresh 是消融实验：100 days 后 SPANN+ P99.9 涨到 >10ms，SPFresh 稳定 ~4ms。

### 与 [DiskANN](./diskann.md) 的对比

[xu-2023-spfresh §5.2 Workload A]：

| | DiskANN（streamingMerge）| **SPFresh** |
|---|---|---|
| 算法路线 | Graph + 周期 rebuild | IVF + LIRE in-place |
| 100 days P99.9 | 飙到 >20ms during rebuild | **稳定 ~4ms** |
| 100 days P99.9 平均 | baseline | **2.41× 更低** |
| 1B rebuild 资源 | 1100 GB + 32 cores × 2d | **持续 10 GB + 2 cores** |
| 内存常驻 | PQ codes + graph cache | centroids + version map |
| Recall 趋势 | 跌后 rebuild 复原 | 稳定甚至缓涨 |

详见 [SPFresh vs DiskANN/SPANN+ benchmark](../benchmarks/spfresh-vs-diskann-spann-update.md)。

### 与 [Milvus](./milvus.md) 的关系

Milvus v2.6.x 的 LSM segment 模型是另一种 dynamic data 解——周期 merge 而非 in-place。两条路线对偶：

- **Milvus**：DBMS 层处理动态；segment 内仍可用静态算法
- **SPFresh**：算法层处理动态；NPA 主动维持

**理论上**：SPFresh 算法可作为 Milvus 一个 segment 内 index——但 Milvus 当前不集成 SPFresh。

## 生产案例

[xu-2023-spfresh] 论文不直接给 Microsoft Bing 等生产部署数据。但作者团队（Qi Chen 等）正是 [SPANN](./spann.md) @ Bing 的同一组——SPFresh 大概率会回流到 Bing 替代 SPANN 周期 rebuild 路径，但论文未明示。

### Turbopuffer — 首个 OSS-known production deployment（frontier closed, 2026-05-11）

[per sources/docs/turbopuffer/llms-full.txt §architecture §concepts §vector]

> Vector indexes are based on [SPFresh](https://dl.acm.org/doi/10.1145/3600006.3613166). SPFresh is a centroid-based approximate nearest neighbour index... A centroid-based index works well for object storage as it minimizes roundtrips and write-amplification, compared to graph-based indexes like HNSW or DiskANN. — Turbopuffer architecture docs

[Turbopuffer](./turbopuffer.md) 是 wiki 内**首个 OSS-known production deployment** of SPFresh。Microsoft Bing 内部部署 implied 但 closed；Turbopuffer 公开声明 SPFresh 作为唯一 vector index 算法 + 公开 production scale numbers（3.5T+ docs / 100B+ vectors queryable / 100M+ namespaces）。

**Frontier closure timeline**：
- 2021 — Microsoft SPANN paper (chen-2021-spann)，centroid-based ANN with rebuild
- 2023 — Microsoft SPFresh paper (xu-2023-spfresh)，加 LIRE protocol 实现 in-place update
- 2026 — Turbopuffer commercial SaaS production 公开声明 SPFresh as primary ANN——**仅 3 年从学术论文到 commercial production deployment**

**Turbopuffer 给 SPFresh 的关键 adaptation**（具体 fork 程度闭源未公开）：
- 原 paper 用 SPDK + raw SSD；Turbopuffer 跑在 **object storage** (S3/GCS) 之上——roundtrip 成本数量级不同（~100ms vs ~10µs）
- 用 Rust 重写（paper 是 C++ + SPDK）
- 集成 object-storage-native WAL + LSM tree 而非 paper 的 SPANN-style centroid + posting list

**为什么 SPFresh 适合 Turbopuffer 哲学**（Turbopuffer docs 论点）：
- Graph-based ANN (HNSW / DiskANN): traversal 需 ~log(N) roundtrips → cold query × 100ms per roundtrip 太慢
- Centroid-based (SPFresh): 1 read centroid index + 1 batch fetch posting lists → minimal roundtrips → object-storage-friendly
- LIRE protocol incremental update → 不需 rebuild → fit Turbopuffer 持续 writes 哲学

**Open: Turbopuffer SPFresh adaptation 是否破坏 LIRE NPA**：原 LIRE 假设 raw SSD low-latency；object storage roundtrip × 100ms 下 LIRE rebalance trigger 频率是否仍可控？docs 未明示。详见 [systems/turbopuffer.md "Open Questions"](./turbopuffer.md)。

## Open Questions

- **分布式版本**：[xu-2023-spfresh §6] 明示 future work；多 SSD / 多机的 cross-shard LIRE 协议未设计
- ~~**Graph-based 适配**：LIRE 仅适用 cluster-based；HNSW / Vamana 类 graph 上的 in-place update 仍开放~~ **2026-05-09 ingest [singh-2021-freshdiskann] 已答**：[FreshDiskANN](./freshdiskann.md) 用 [FreshVamana](../concepts/freshvamana.md) (α=1.2) 给出 Vamana graph 的 streaming 解；SPFresh 与 FreshDiskANN 是 cluster-path / graph-path **姐妹工作**，都来自 Microsoft Research。**Memory budget 差异**：SPFresh ~4 GB 1B vs FreshDiskANN ~128 GB 1B（30× 差距）——cluster path 更经济。HNSW 因隐式 α=1 仍未解；NSG 同。详见 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md) 的"In-place 的两条路径"
- **MIPS 任务**：LIRE 与 SPANN 同样未在 MIPS 验证
- **大 update batch**：1% daily 假设；burst 写场景（10% / 50% daily）下 LIRE 触发率与 cascading 上限未量化
- **冷热极端 skew**：所有插入集中到少数 hot region 时 LIRE split 频繁度
- **与 [Vamana](../concepts/vamana.md) / [DiskANN](./diskann.md) 的混合**：DiskANN 的 graph + SSD re-rank 思路与 SPFresh 的 IVF + LIRE 是否可融合？开放方向
- **Updater 与 search 干扰**：实验单 NVMe SSD；多 NVMe 时 SPFresh 的资源平衡可能不同
- **OPQ / [ScaNN](../concepts/scann.md) 等量化在 SPFresh 下**：当前 LIRE 假设 posting 全精度（继承 SPANN）；融合量化是否破坏 NPA？

Cited by: 待 query 引用
