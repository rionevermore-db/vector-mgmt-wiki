---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [systems/milvus.md, concepts/delta-consistency.md, concepts/manu-ssd-hierarchical-kmeans.md, benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md, systems/spfresh.md, systems/spann.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 8
---

# Post-snapshot (guo-2022-manu): giga-scale-sharding

## TL;DR (delta from pinecone-docs post)

**Manu (Milvus 2.x 学术论文) 实证 24h auto-elasticity + 2-10 query nodes 线性扩展**——这是 wiki 现有 source 中**首次 elasticity 实测**。给定 16 × 1TB 私有云：Manu 路线（= Milvus 2.x = 已有推荐的 Milvus 路径）现在有学术论文背书。**Delta consistency τ** 让"周级 update"约束变成"调 τ"——若用户能容忍 ≤τ 的 staleness，写完几乎立即可见，无需双索引切换。

## Answer

### Manu elasticity 实测的工程含义（NEW）

[per benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md "Elasticity Test"]

Manu §5.2 在 24h 真实 e-commerce workload 上实测：
- search latency < 100ms → reduce query nodes 0.5×
- search latency > 150ms → add query nodes 2×

**16 节点私有云的对应推论**：
- 白天峰可以全 16 节点跑 query
- 夜晚谷可以缩到 ~4-8 节点（K8s pod 缩减）
- 促销大峰自动 scale up

→ Manu 在 cluster 层级做的 elasticity 是 **私有 K8s 集群**可直接复制的——同样的 K8s + 同样的 Milvus 2.x 软件栈。

### Manu scalability 实测的延伸（NEW）

[per benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md "Scalability Tests"]

- **Query nodes 2-10 线性扩展**（HNSW + IVF-FLAT 都是）
- **Dataset 20M-100M 倒数关系**（per-vector 工作量基本固定）
- **关键结论**：用更大 segment 可缓解大 dataset throughput 下降

**16 节点 × 1TB 千亿场景推论**：
- 16 节点在 100M scale 已实测高 throughput；千亿 = 100M × 1000 = 数据量增加 1000×
- 单节点 throughput 倒数下降 → 需横向扩展
- 16 节点 + 大 segment 配置预期 throughput **0.1-1× of 100M baseline**——分秒延迟可达，<50ms 难

→ 千亿规模下 Manu 实测路径的**单节点处理能力是 NeurIPS 2021 winner SSD index 关键**——比 baseline same QPS recall 高 60%，意味着同 throughput 下 recall 上一个台阶，间接给出 latency budget。

### Delta Consistency 的工程含义（NEW）

[per concepts/delta-consistency.md]

之前 query 假设的"周级别全量更新"是 batch-style；Manu 提供更细粒度方案：

```
设 τ = 10s（用户可调）：
  - 写入实时进 memtable + 写 WAL + LSN
  - Subscriber（query node）持续消费 time-tick
  - Query 检查 L_r - L_s < τ；若 unsatisfied 等待
  - 写入后最多 τ 时间即对 query 可见
```

**对周级"全量更新"约束的影响**：
- 如果用户能接受 τ = 10s（典型 e-commerce）：增量数据 10s 内可见，**完全不需要 batch 全量切换**
- 如果用户必须 strong consistency（金融、风控）：τ = 0，每 query 等所有写入 → latency 高
- 如果可以 eventual：τ = ∞，throughput 最大

→ Manu 把"周级 batch update" 重新表述为"持续小批量 + τ 控 staleness"——更灵活。

### 16 × 1TB 私有云的最终推荐（updated）

| 维度 | 推荐 | 论文 / 文档 |
|---|---|---|
| 部署 | Milvus 2.6.x（Manu 架构演化产品） | [systems/milvus.md] |
| Index | HNSW（per-segment）或 DiskANN（SSD 节省内存） | [milvus-docs] / [Manu §4.4 SSD 路线] |
| Update strategy | Stream indexing + delta consistency τ=10-60s | [concepts/delta-consistency.md] |
| Elasticity | K8s + 自定 latency thresholds | [Manu §5.2 实测] |
| 千亿 throughput | 增大 segment + 多 query nodes 线性扩展 | [Manu §5.2 scalability] |
| 万亿 | wiki 实测案例只到 1.5T (Meta Faiss mmap) ~1s 延迟 | 未实证可达 P99<50ms |

→ **建议方向**：千亿宽松；万亿不可行（受 wiki 现有 source 限制）。Manu 实测仅到 100M。

### 已知盲区

- **Manu 1B+ 实测**：论文未给（Manu §5 只到 100M）；实际 Milvus v2.6.x 应支持但 wiki benchmark 未覆盖
- **Manu SSD index 在生产 Milvus 中的状态**：[per concepts/manu-ssd-hierarchical-kmeans.md "Open Q"] v2.6.x 文档没显式列此 index——可能被 DiskANN 集成取代
- **Pinecone vs Manu 实测对比**：双方都不公开
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo

## Cited Pages

- [systems/milvus.md](../../systems/milvus.md)
- [concepts/delta-consistency.md](../../concepts/delta-consistency.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
- [benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md](../../benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/spann.md](../../systems/spann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
