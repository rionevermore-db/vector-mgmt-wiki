---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [systems/diskann.md, concepts/filtered-vamana.md, benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md, systems/spann.md, systems/spfresh.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (gollapudi-2023-filtered-diskann): giga-scale-sharding

## TL;DR (delta from yang-2020-pase post)

**Filtered-DiskANN 实测仅到 28M（DANN dataset）**——不直接覆盖千亿规模。但论文证明 **filter-aware Vamana + DiskANN SSD 框架** 在 28M 数据 thousand-QPS @ 90%+ recall 可行——**外推到千亿**需多机分片 + per-shard FilteredVamana + 应用层 routing。**对 read-heavy + 周级 update + filter-heavy** workload，FilteredVamana incremental 友好（streaming-add）—— 比 [SPANN](../../systems/spann.md) static + 周期 rebuild 路径在 update 维度更优。

## Answer

### Filtered-DiskANN 与千亿规模的差距（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "SSD 模式"]

论文实测最大 SSD scale：
- **28M DANN dataset, 1pc-100pc specificity**
- 24 threads, beam width 4, search L 40-100
- thousand-QPS @ 90%+ recall

→ 千亿规模需要 1000× 数据量缩放——论文未实测。但 [DiskANN](../../systems/diskann.md) 已实证 1B SIFT 单机；filter-aware 版本理论上同 scale-up。

### 给定 16 节点的部署推断（NEW）

如果 workload 包含大量 filter（典型推荐 / 广告系统 region / 类目 / 时间）：

```
16 节点 × 1TB RAM，每节点：
  - DiskANN-style hybrid: PQ codes in DRAM + FilteredVamana graph on SSD
  - 千亿数据 → ~ 6.25B per node
  - 6.25B × 768d × float32 = ~19 TB → 需 SSD 路径
  - FilteredVamana edges + PQ codes → DRAM ~10-50 GB / 节点
  - 应用层按 filter 集合 / 数据 ID 分到 16 节点
```

**关键**：FilteredVamana **支持 incremental insert**——周级 update 不需要全 rebuild。这与 [SPANN](../../systems/spann.md) static + 周期 rebuild 路径不同：
- SPANN: 周期重建 centroids；update 期间老 index 服务
- FilteredVamana: 持续增量加点；不需要切换

→ **read-heavy + 持续 update**（query 像广告系统，新广告随时进入）场景，FilteredVamana 比 SPANN 更友好。

### 关键 Production 论据：Microsoft 广告 A/B（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "Microsoft 广告 A/B 实验"]

Microsoft 实验直接对应"region-filter heavy"场景：
- 47 region filters
- A/B test 2 周
- **+34.61% clicks, +48.95% revenue**
- 小 region (<1% share, 27 个): **+70%/+80%**

→ 16 节点千亿场景若 workload 类似（按地理 / 类目 / 用户分群 filter），FilteredVamana 比 search-time filter 系统（Milvus 5-strategy / Pinecone hybrid）有显著生产价值。

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| Filter-heavy workload 解 | Milvus 5-strategy / ADBV 4-plan / PASE iterative | **+ FilteredVamana filter-aware build**（首个 build-time 方案） |
| 千亿+ filter scale | 未实证 | 28M 实证 + 推断千亿可行 |
| Production A/B test 数据 | 无 | **+34.61% clicks Microsoft 广告**（P=0.03）|
| Update 模型 | 各异 | **FilteredVamana incremental 友好** |

### 已知盲区

- **Filtered-DiskANN 千亿规模实测**：仅 28M
- **多 filter conjunction（AND/OR/NOT）**：论文 §2 明确"future work"
- **跨 16 节点的 filter-aware routing**：论文未涉及
- **2026 SIGMOD 跨 model 整合**：talk 当日 demo

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md](../../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
