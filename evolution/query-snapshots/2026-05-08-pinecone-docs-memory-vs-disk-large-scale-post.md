---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: pinecone-docs
wiki-pages-total: 36
cited-pages: [systems/pinecone.md, concepts/pinecone-serverless-slabs.md, systems/spann.md, systems/diskann.md, systems/spfresh.md, systems/milvus.md, topics/disk-vs-memory-ann.md]
cited-count: 7
---

# Post-snapshot (pinecone-docs): memory-vs-disk-large-scale

## TL;DR (delta from xu-2023-spfresh post)

**新增第 6 条路线 + adaptive indexing 维度**：[Pinecone Serverless](../../systems/pinecone.md) 把 [SPANN](../../systems/spann.md) / [Milvus 2.x](../../systems/milvus.md) 已用的"object storage + memtable" 模式**与 adaptive indexing 结合**——slab merge 不只合并数据，**还升级索引算法**。这是 wiki 内首个"索引随生命周期演化"系统。私有部署同等思路可借鉴但需自己实现。

## Answer

### 路线对比表完整版（updated with Pinecone）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 动态数据 | 索引演化 |
|---|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ | 静态 |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ | 静态 |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ | 静态 |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ | 静态 |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB peak | rebuild | 静态 |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB peak | 周期 rebuild | 静态 |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ | 静态 |
| DBMS LSM segment | DRAM + S3 | — | ~64 GB/节点 | LSM merge | 静态（segment 内） |
| DBMS cloud-native + zero-disk WAL | etcd + S3 | — | stateless | LSM + Streaming Node | 静态 |
| 磁盘 + IVF + 全精度 + In-place 增量 | DRAM + raw NVMe | >0.86 @ ~5 ms | 持续 10 GB | LIRE in-place | 静态 |
| **SaaS + slab + adaptive indexing**（NEW） | Memtable + cache + slab on object storage | docs 未公开 | docs 未公开（auto） | **slab merge** | **adaptive across lifecycle** |

### 第 5 维度：索引随生命周期演化（NEW，本次 ingest 暴露）

之前 disk-vs-memory 4 维度：(1) recall, (2) latency, (3) memory cost, (4) update strategy。
**Pinecone 添加第 5 维度：索引随生命周期演化**。

[per concepts/pinecone-serverless-slabs.md "Layer 2：Slab"]

```
namespace lifecycle:
  ↓ 起步：少量小 slab → fast indexing
  ↓ 成熟：merge 创大 slab → sophisticated indexing
  ↓ 长期：进一步合并 → 更精确算法
```

→ 单 namespace 内**多代算法并存** + query router 透明合并。这是 wiki 第一次见的"索引算法在线演化"模式。

### 三层"磁盘 vs 内存"反转的最新一层

[per topics/disk-vs-memory-ann.md]：

之前已有：
1. vector data DRAM→SSD（DiskANN/SPANN）
2. write path memory→segment（Milvus LSM / Pinecone slab）
3. WAL broker disk→object storage（Woodpecker）

**Pinecone 在第 1+2 层基础上加：**
- Slab 写到 **object storage**（S3 等，**远比 NVMe 便宜慢**）
- Cache 在 memory + local SSD（与 SPANN 思路同）
- 冷 slab 第一次访问从 object storage fetch（cold start latency 是 trade-off）

**Pinecone DRN 模式不同**：把所有 slab 全部 cache 到 dedicated 节点的 memory + local SSD（"guaranteed warm"）——把 object storage 当持久层，缓存层完全隔离。

### Pinecone vs SPANN/Milvus 的三角关系（NEW）

| | SPANN | Milvus 2.x | **Pinecone Serverless** |
|---|---|---|---|
| Posting / segment 存储 | NVMe SSD（local） | S3 / HDFS | **Object storage（S3/GCS/Azure）** |
| 内存 cache | centroids + SPTAG | 节点 memory（Streaming Node） | memory + local SSD（query executor） |
| 写路径 | 静态（rebuild） | LSM segment | **memtable → flush → slab + adaptive index** |
| 索引算法 | IVF + closure | segment 内固定 | **adaptive across slab lifecycle** |
| 冷启动 latency | n/a | n/a | docs 未公开（best-effort cache） |
| Multi-region | ✗ | ✓ | ✓ |

### 已知盲区

- **Pinecone billion-scale 实测数字**：[per systems/pinecone.md "Open Q"] docs 仅说"scalability to billion-vector datasets"，无具体配置 / 延迟 / recall
- **Pinecone DRN cold start P99**：当所有 slab 都 cache 到 dedicated nodes 时是 0；on-demand 时未公开
- **Adaptive indexing 实际效果**：vs Milvus 固定 index_type 的对比未量化
- **Pinecone vs SPFresh 性能**：两者都 cluster-based + object/SSD storage，但更新策略不同（slab merge vs LIRE in-place）；实测对比仍是 wiki 整体盲区

## Cited Pages

- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/pinecone-serverless-slabs.md](../../concepts/pinecone-serverless-slabs.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/milvus.md](../../systems/milvus.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
