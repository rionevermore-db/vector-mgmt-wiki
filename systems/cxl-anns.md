---
title: CXL-ANNS（CXL 内存解耦 + 近数据计算的 billion-scale ANN 系统）
type: system
sources: [jang-2023-cxl-anns]
related: [distributedann.md, diskann.md, spann.md, faiss.md, cagra.md, ../concepts/distributedann-cited-frontier.md, ../concepts/product-quantization.md, ../concepts/nsg.md, ../concepts/hnsw.md, ../concepts/cagra-graph.md, ../topics/disk-vs-memory-ann.md, ../topics/gpu-vs-cpu-ann.md, ../topics/index-selection.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/spann-vs-diskann-billion.md]
created: 2026-05-19
updated: 2026-05-19
---

# CXL-ANNS

**TL;DR**: KAIST + Panmnesia 在 **USENIX ATC 2023** 发表的 **software-hardware collaborative** billion-scale ANN 系统——把**全部** billion-point 数据集（graph + embedding table，不压缩、不下放 SSD）放进 **CXL（Compute Express Link）解耦内存池**，从而 billion-scale **不损失任何精度**，同时比"无限 DRAM 的 oracle 系统"还**快 3.8× throughput / 低 68% latency**。核心矛盾：naive CXL 内存池是"far-memory"——每次访问要 CPU↔flit 协议转换，比 oracle **慢 3.9×**；CXL-ANNS 用 4 个机制消除这个 gap：(1) **relationship-aware graph caching**（按 entry-node hop 距离把热点 2-3 跳内的 node 缓存进本地 DRAM）；(2) **CXL-aware + ANNS-aware prefetching**（82.3% 的访问来自 candidate array → 提前一轮预取）；(3) **EP-side 近数据距离计算 + vector sharding**（domain-specific accelerator 在内存侧算距离，数据传输减 73.4×）；(4) **urgent/deferrable 依赖松弛 + 细粒度调度**（CXL CPU 不再 42% 时间空等）。实测 **111.1× higher QPS / 93.3% lower latency vs SOTA billion-scale 方法**（PQ 压缩 / DiskANN / HM-ANN）。有 **16nm FPGA + RISC-V 真实原型** + gem5 全系统仿真。是 wiki 内**首个 hardware-level memory disaggregation ANN 系统**，也是对 [CAGRA](./cagra.md) GPU-native 路线的**显式哲学反论**（§7：GPU 对 ANN 距离计算反而不经济）。[jang-2023-cxl-anns]

> **本 page 关系**：CXL-ANNS 此前仅作为 4 篇 bundled 论文之一在 [concepts/distributedann-cited-frontier.md](../concepts/distributedann-cited-frontier.md) 占 ~10 行二手摘要（2026-05-12 batch ingest）。本 page 是 2026-05-19 deepen-ingest——从一手 USENIX ATC 2023 论文（17 页）提取真实技术深度。bundled page 的 CXL-ANNS 段已收窄为指针。

## 提出背景

[jang-2023-cxl-anns §1, §2.2, §3.1]

Billion-scale ANNS 需要巨量 DRAM——论文举例：Microsoft Bing/Outlook 100B+ 向量 × 100 维 > **40 TB** 内存；Alibaba 电商 2B+ 向量 128 维需 TB 级。单机 DRAM 装不下，此前两条路线**都牺牲精度或性能**：

| 路线 | 代表 | 机制 | 论文给出的代价 |
|---|---|---|---|
| **Compression** | PQ-style 量化 | 把 embedding table 量化（用 cluster centroid 替原向量） | 精度随压缩率上升而显著下降；论文实测 45.8% 数据缩减后**够不到 0.9 recall@k**（Yandex-D 仅 0.58）[§3.1 Fig 5] |
| **Hierarchical** | [DiskANN](./diskann.md) / HM-ANN | graph + 全向量放 SSD/PMEM，低精度搜索（压缩）+ 高精度 re-rank（SSD） | storage 访问 = 总 query latency 的 **87.6%**；DiskANN / HM-ANN latency 比 oracle 差 **29.4× / 64.6×** [§3.1 Fig 6] |

> **CXL-ANNS 的主张**：不要压缩、不要下放慢存储——直接把全量数据集放进 CXL 解耦内存池，billion-scale **无精度损失**，并把 CXL 的 far-memory 延迟用软硬件协同**藏掉**，最终比 oracle（无限本地 DRAM）还快。"Oracle" 在本论文指拥有不受限 DRAM 容量、纯内存的系统。

### 为什么 naive CXL 不够：far-memory 问题

[jang-2023-cxl-anns §2.3, §3.1]

CXL 是基于 PCIe 物理层的开放互连标准，3 个 sub-protocol：

| Sub-protocol | 作用 |
|---|---|
| **CXL.io** | ≈ PCIe，EP 枚举 + transaction 控制 |
| **CXL.cache** | 让 EP 与 host CPU cache 保持一致 |
| **CXL.mem** | 简单 load/store over PCIe（内存扩展用） |

3 种 endpoint（EP）类型：Type-1（co-processor，仅 CXL.cache）、Type-2（带内存的 accelerator，CXL.cache+CXL.mem，如 GPU/FPGA）、**Type-3（纯内存扩展器，仅 CXL.mem read/write——CXL-ANNS 用这个）**。Type-3 EP 内部内存暴露为 **host-managed device memory (HDM)**，映射进 CXL CPU 的 **host physical address (HPA)**，应用用普通 load/store 访问。RC（root-complex）把内存请求转换成 CXL packet（**flit**），交换式网络支持最多 **4095 个 CXL EP / 内存池可达 4 PB**。

**问题**：HDM 的每次访问都要 RC 做 memory→flit 转换 + 反向 flit→memory——这条 far-memory 路径让 naive CXL 内存池版 ANNS（论文称 `Base`）比 oracle **慢 3.9×**（graph traverse +2.6×、distance calc +4.3×）。CXL-ANNS 的全部工程都在消除这个 3.9×。[§3.1, §6.2]

## 三个驱动设计的实测观察

[jang-2023-cxl-anns §3.2]

| # | 观察 | 数据 | 导出的机制 |
|---|---|---|---|
| 1 | **Node-level relationship**——所有 graph traverse（BFS, Algorithm 1）都从单一 entry-node 出发；靠近 entry-node 的 node 被访问得远多于外层 | 1M kNN query 中最频繁访问的 node 落在 entry-node **2~3 edge hop** 内 [Fig 9b] | → relationship-aware caching：把低 hop-count node 放本地 DRAM |
| 2 | **Distance calculation 主导** | 4 个子任务（candidate update / memory access / distance calc / graph traverse）中 distance calc 占执行时间 **81.8%**；data vector 比 graph data 大 **2.0×**（高维）；但 distance calc 计算量低 → 适合 HW 加速 [Fig 10] | → EP-side DSA 近数据距离计算 |
| 3 | **Vector reduction** | ANNS 只要 scalar 距离值，不要完整向量 → 若 EP 在内存侧算距离，传输量按每向量维度降低 → 平均**少传 73.4× 数据** [Fig 11b] | → vector sharding + 近数据计算 |

## 架构图

[jang-2023-cxl-anns §3.3, §4, §5, Fig 12]

```
┌─ RC-side（CXL CPU host）──────────────────────────────┐
│  S/W stack:                                            │
│   ├─ Query Scheduler  ── 把每 query 拆 3 子任务:        │
│   │     graph traverse / distance calc / candidate upd │
│   │     graph traverse + candidate update 留 CPU 侧     │
│   │     distance calc 下发给 EP（协同 pool manager）    │
│   ├─ Pool Manager     ── 管 HDM 的 HPA 映射;            │
│   │     按 edge-hop count 决定 node 放本地 DRAM 还是池  │
│   └─ Kernel Driver    ── 枚举 EP, 把 HDM 映射进 HPA;    │
│         EP 接口寄存器映射到 RC PCIe 地址（用 CXL.io）   │
│  Local DRAM: 缓存 2-3 hop 内热点 node 的 graph+vector   │
├─ CXL RC ── memory⇄flit 转换 ── CXL switch ─────────────┤
│  CXL-ANNS EP-side（×4，每 EP 4 memory controller）:     │
│   ├─ PHY controller + CXL engine（flit⇄memory）        │
│   ├─ DSA（domain-specific accelerator）:                │
│   │     多个 PE = adder/subtractor + multiplier 算术树  │
│   │     L2 vs angular 距离按 encoding 路由              │
│   └─ DRAM modules（HDM）：embedding table 列向 sharding │
│         round-robin 分布到各 EP                         │
└────────────────────────────────────────────────────────┘
   内存池可扩展：1 CXL switch → 多 Type-3 EP（≤4095, ≤4PB）
```

## 数据流 / 控制流

### Index 准备（pool manager，§4.1-4.2）

[jang-2023-cxl-anns §4.1, §4.2, Fig 13, Fig 14]

```
1. Kernel driver 在 PCIe 枚举期发现 CXL Type-3 设备，
   从 device tree / ACPI 读 RC base，把每个 HDM 连续映射进 HPA
   → 形成 "CXL arena"（per-arena 连续虚拟地址空间，可区分各 EP）

2. Pool manager 跑 SSSP（single-source shortest path）：
   从固定 entry-node 出发，BFS 式给每个 node 算 edge-hop count
   （多线程加速；构造完线程终止，不占 CPU）

3. 按 hop-count 升序排序，把 hop 最小的 node（graph 邻接表 + feature
   vector）尽量塞进本地 DRAM（容量经 sysconf _SC_AVPHYS_PAGES ×
   _SC_PAGESIZE 估计）；其余放 CXL 内存池

4. Pool manager 在 CXL arena 内双向分配：
   - data vectors（定长，大）→ stack-like allocator（向上增长）
   - graph 邻接表（16B~1KB 变长）→ buddy-like allocator（向下增长）
   - embedding table 列向 sharding，round-robin 跨多 EP（vector sharding）
```

### Query 服务（query scheduler，§5）

```
对每个 kNN query（best-first search，Algorithm 1）：
  循环直到无新候选:
    ① graph traverse（CPU 侧）—— 取 candidate array 中未访问 node
    ② distance calc（下发 EP）—— DSA 在内存侧算 query↔neighbor 距离:
         - embedding table 列向 sharding → 每 EP 算自己那段 sub-distance
         - 各 PE 终端并行从 4 个 DIMM channel 读，算术树累加
         - CXL CPU 把各 EP 的 sub-distance 累加成最终距离
         - 接口走 CXL.io memory-mapped 寄存器（doorbell + command buffer）
    ③ candidate update（CPU 侧）—— 插入/排序 kNN 候选 + 选下一 node
  prefetch: query scheduler 提前一轮预取下一迭代要访问的 node
            （speculation 依据：82.3% 访问来自 candidate array）
  调度: 把 ② 的等待时间用于执行 ③ 中 deferrable 的部分（细粒度）
```

## 关键设计决策

### 1. Relationship-aware graph caching（§4.1）

[jang-2023-cxl-anns §4.1, Fig 13]

**决策**：用 entry-node 的 edge-hop count（SSSP 算出）而非访问频率统计来决定缓存优先级。
**Trade-off**：
- 优势：免运行时 profiling；hop-count 是静态图结构属性，构造一次即可；与"BFS 从固定 entry-node 出发"的 ANNS 本质对齐——观察 1 证明热点确实集中在 2-3 hop 内
- 代价：大图（如 Meta-S，256 维 190 邻居，graph 远超本地 DRAM）只有 **13.8%** 的 graph traverse 能本地命中（小图如 BigANN/Yandex-T 达 **92.0%**，因 graph 仅 ~129.3 GB）——大图必须靠 prefetch 补救

### 2. CXL-aware + ANNS-aware prefetching（§5.2）

[jang-2023-cxl-anns §5.2, Fig 17, Fig 18]

**决策**：query scheduler 提前一个 ANNS 迭代预取下一轮要访问 node 的 graph 信息，藏住 CXL far-memory 延迟。
**关键 insight**：预取需要知道"下一轮访问哪些 node"，但 ANNS 有 procedural 数据依赖（要先算距离才知道）。论文观察 **82.3% 的总访问 node 来自 candidate array**（即使该 array 还没为下一步更新）→ 据此 speculative 预取。
**实测**：单纯本地 caching（NoPrefetch）每次 cache miss 走慢 CXL = 75.4 ns，缩短 45.8%，但大图（Meta-S/MS-S）仅 24.5% 可缓存 → 仍比 oracle 高 2.3×；加 prefetch（Cache）→ 延迟缩 8.5×，**比 oracle 还短**；Meta-S 上藏掉 CXL 延迟 **72.8%**。

### 3. EP-side 近数据距离计算 + vector sharding（§5.1）

[jang-2023-cxl-anns §5.1, Fig 15, Fig 16]

**决策**：不把向量搬回 CPU 算距离，而在 Type-3 EP 内置 **DSA（domain-specific accelerator）**——多个 PE，每个是 adder/subtractor + multiplier 的算术逻辑树；L2 距离走 subtractor，angular 距离走 multiplier（按 dataset encoding 路由）。
**Vector sharding**：单 EP 的 PE 多但单 EP backend DRAM 带宽会成瓶颈（每向量 ~256 维 ~1KB）→ 把 embedding table **列向切分**，按每 EP 的 I/O 粒度（256B）round-robin 分到各 EP；每 EP 并行算自己那段 sub-distance，CXL CPU 累加得最终距离（L2/angular 都是 element-wise 累加，可分布）。
**Trade-off**：引入 RC↔EP 接口开销（doorbell / command buffer / CXL.io memory-mapped 寄存器），但仅占 Base 距离计算延迟的 5.4%。
**实测**：distance calc 时间降 **119.4×**，query latency 比 Base 低 **7.5×**、比 oracle 低 **1.9×**（oracle 有无限 DRAM 但不懂 ANNS 行为 → data movement 开销大）；data vector 传输削 **21.1×**。

### 4. Urgent / deferrable 依赖松弛 + 细粒度调度（§5.3）

[jang-2023-cxl-anns §5.3, Fig 19, Fig 28]

**问题**：ANNS 计算序列串行依赖；EP 算距离时 CXL CPU 空闲——Yandex-D 上 CPU **42%** 时间在干等距离结果。
**决策**：把 candidate update 拆成 **urgent**（node selection——下一轮 graph traverse 必须用）vs **deferrable**（candidate array 插入/排序——下一步未更新前不必做）。query scheduler 在 graph traverse 前先做 node selection（urgent），把 deferrable 部分延迟到 distance-calc 等待窗口内执行。
**实测**：idle 减 1.3×，QPS 比 Cache 再 +15.5%（candidate array 重叠 16.3× 放大）。

## Scale 边界

[jang-2023-cxl-anns §2.3, §6.4]

| 配置 | 数据 | 实测 / 推断 |
|---|---|---|
| **单 CXL host + 4-8 Type-3 EP** | 6 个 1B 数据集（128-256 维） | 论文主实验配置，all-in-memory 无精度损失 |
| CXL 内存池上限 | — | 交换式网络 ≤ **4095 EP / ≤ 4 PB**（架构上限）[§2.3] |
| **Bigger dataset（Yandex-D ×4 → 4B）** | 合成 +3B 噪声向量 | CXLA 比 oracle 低 **2.7× latency**；EP 数不变时 PE 成瓶颈，需加更多 EP 分摊计算 [§6.4 Fig 29] |
| **Multi-host（多 CXL CPU 分担）** | embedding table 跨 host 分区 | QPS 随 host 数上升直到 **4 host**；**6 host 时 QPS 下降**——PE 数不足成瓶颈（doorbell 等无空闲 PE），加 EP 可解 [§6.4 Fig 30] |

> **wiki 解读**：CXL-ANNS 受 **CXL 内存池物理容量**约束（4 PB 是协议上限，实际由 EP 数 × 单 EP DRAM 决定），不受单机 DRAM 约束——这是与 [DiskANN](./diskann.md)（64 GB RAM + SSD）/ [SPANN](./spann.md)（centroid in-DRAM + posting on-SSD）完全不同的 budget 维度：**CXL-ANNS 用"更多内存设备"换 scale，DiskANN/SPANN 用"更慢存储介质"换 scale**。瓶颈不在容量而在 EP-side PE 算力——故 scale-out 方式是"加 EP/host"而非"加 SSD"。

## 与 wiki 已有系统的对比

### 与 [DiskANN](./diskann.md) / HM-ANN（hierarchical SSD 路线，论文的 baseline）

[jang-2023-cxl-anns §3.1, §6.2]

| | DiskANN / HM-ANN | **CXL-ANNS** |
|---|---|---|
| 数据驻留 | graph + 全向量在 SSD/PMEM；PQ 在 DRAM | **全量（graph + embedding）在 CXL 内存池，不压缩** |
| 精度 | 高精度 re-rank 才达标 | **无精度损失**（全精度全在内存） |
| 瓶颈 | storage 访问 = 87.6% query latency | CXL far-memory（被 caching+prefetch 藏掉） |
| vs oracle latency | DiskANN 差 29.4× / HM-ANN 差 64.6× | **比 oracle 还低 68%** |
| scale 方式 | 加 SSD 容量 | 加 CXL EP/host |

→ CXL-ANNS 的论点：hierarchical 路线把"慢"藏在 storage 层换不到真低延迟；CXL 内存池 + 软硬协同可以**既要 billion-scale 又要 full precision 又要低延迟**。

### 与 [DistributedANN](./distributedann.md)（同解 memory bottleneck，路径不同）

[per concepts/distributedann-cited-frontier.md; adams-2025-distributedann §4.3]

两者都解 billion-scale 内存瓶颈，但 hardware 维度正交：

| | DistributedANN (Bing 2025) | **CXL-ANNS (KAIST 2023)** |
|---|---|---|
| 路径 | single DiskANN graph 跨 1000+ 机器，distributed KV store 作 shared-disk | **单 CXL host + 解耦内存池**（多 Type-3 EP），near-data 距离计算 |
| 扩展粒度 | 加机器（distributed） | 加 CXL EP/host（disaggregated memory，单 cluster 内） |
| near-data 计算 | node scoring service 跑在 KV host（~6× bandwidth saving） | DSA 跑在 Type-3 EP（distance calc 119.4× 加速） |
| 商用现状 | Bing production（已替换 SPANN-style） | KAIST + Panmnesia 研究原型（16nm FPGA），无公开 vendor production |

→ **共同主题**：都用 "near-data computation" 削 ANN 的内存带宽瓶颈，但 DistributedANN 是分布式 scale-out，CXL-ANNS 是单 cluster 内存解耦——实际产线选哪种取决于是否有 CXL 2.0+ 硬件。

### 与 [CAGRA](./cagra.md)（GPU-native 路线，§7 显式反论）

[jang-2023-cxl-anns §7]

CXL-ANNS 论文 §7 **明确论证 GPU 不适合 billion-scale ANN**，这与 [CAGRA](./cagra.md) 的 GPU-native 命题正面冲突，是 wiki 内首次出现的两条 hardware 路线**哲学对撞**：

| 论点 | CXL-ANNS §7 立场 | CAGRA 立场 |
|---|---|---|
| 数据移动 | GPU 需与 host SW/HW 层交互 → data transfer 开销不可避免 | GPU 内 dataset 驻留 device memory，避免 host 往返 |
| 距离计算硬件 | ANNS 距离计算用"少量简单 lightweight 向量单元"即可 → GPU 对此**不经济** | GPU 高并行 + 高带宽正适合 batch 距离计算 |
| 内存容量 | CPU+GPU 内存**不可能装下整个 billion-scale ANNS 数据+任务** | 单 A100 ~100M（float32）；更大需 multi-GPU / PQ |
| 结论 | near-data（CXL EP 内算距离）比 GPU 卸载更省数据移动 + cache 层级利用 | GPU-native graph（fixed degree + 非分层）让 graph ANN 在 GPU 上也最优 |

→ **wiki 解读**：这不是"谁对谁错"，而是 **workload + 硬件可得性决定路线**。CAGRA 适合 dataset 装得进 GPU memory 的 high-throughput/low-latency 静态索引；CXL-ANNS 适合 billion-scale 全精度无损 + 有 CXL 内存池硬件。详见 [topics/gpu-vs-cpu-ann.md](../topics/gpu-vs-cpu-ann.md) 新增的"近数据计算 vs 加速器卸载"对比段。

## 生产案例

[jang-2023-cxl-anns §6.1, 作者归属]

- **研究原型**：16nm FPGA 实现 CXL-ANNS 软硬件（RISC-V BOOMv2 作 CXL CPU，4 个 ANNS EP 各 4 memory controller，CXL switch，Linux 5.15.36，适配 Meta FAISS v1.7.2）；另在 gem5 全系统仿真器复现并 cycle-level 交叉验证（仿真配置模拟 Meta 生产环境服务器）
- **作者归属**：KAIST Computer Architecture and Memory Systems Laboratory + **Panmnesia, Inc.**（CXL IP 公司，Hanjin Choi / Miryeong Kwon / Myoungsoo Jung 双属）——同组另有 "Failure tolerant training with persistent memory disaggregation over CXL"（IEEE Micro 2023）
- **无公开 vendor production**：与 wiki 内 [DistributedANN](./distributedann.md)（Bing 闭源 production）/ [SPANN](./spann.md)（Bing）不同，CXL-ANNS 是学术原型；商用化取决于 CXL 2.0+ 内存解耦硬件市场成熟度

> **wiki 解读**：CXL-ANNS 是 wiki 内**第一个把 hardware substrate（CXL 解耦内存）本身作为 ANN scale 解法**的系统——之前所有 disk-resident 系统（DiskANN/SPANN/Starling/SPFresh）都在 software 层做文章（图布局 / 量化 / 增量），CXL-ANNS 引入的是一个**新硬件层**。这条路线能否进 production 取决于 CXL 内存池硬件可得性，是与 SSD 路线、GPU 路线并列的第三条 scale 轴。

## Open Questions

- **CXL 硬件可得性**：CXL 2.0+ 内存解耦硬件市场成熟度未知；论文用 FPGA 原型 + gem5 仿真，无商用 CXL 内存池实测数据
- **CXL-ANNS vs DistributedANN 实际产线选型**：都做 near-data computation，前者单 cluster 内存解耦、后者 distributed graph；规模天花板与运维代价的实证对比缺失
- **大图 caching 失效**：Meta-S 仅 13.8% graph traverse 本地命中，全靠 prefetch——更大图（4B+）或更高维下 prefetch 命中率是否仍 hold 未充分实测
- **multi-host PE 瓶颈**：6 host 时 QPS 下降需加 EP 解；EP/host/PE 的最优配比闭式未给出
- **CXL-ANNS + 增量更新**：论文是静态数据集（NSG 预构建）；CXL 解耦内存上的 streaming insert/delete（类 [FreshDiskANN](./freshdiskann.md) / [SPFresh](./spfresh.md)）完全 open
- **CXL-ANNS + filter / multi-vector / sparse**：仅 pure dense ANNS；与 wiki 内 filter-aware / hybrid retrieval 联合未涉及
- **CXL tier 与 PMEM/Optane 的关系**：CXL.mem 与 Optane PMEM 都是 DRAM-SSD 之间的中间层；CXL-ANNS 用 CXL 池替代 hierarchical 的 PMEM，但二者作为内存层级的边界 wiki 未系统化
- **embedding model 升级**：与 wiki 全 frontier 一致——zero coverage

Cited by: 待 query 引用
