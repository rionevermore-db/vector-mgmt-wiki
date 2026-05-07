---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, systems/faiss.md, systems/diskann.md, systems/spann.md, systems/milvus.md, benchmarks/faiss-trillion-scale.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, concepts/nsg.md, concepts/product-quantization.md]
cited-count: 10
---

# Post-snapshot: scale-tier-shifts

## TL;DR

Wiki 现给出**四个明确质变点**（新增第 4 点 DBMS 化）：(1) ~10M → 1M+：触发索引必要；(2) **1B 单机 RAM 触顶**：DRAM → SSD（DiskANN/SPANN）或多机分片；(3) **百亿+ → 万亿**：单机 SSD 触顶 → 分布式 mmap；**(4) NEW：从静态算法到 DBMS 形态质变**——任何规模档下，引入持续写入 / 分布式 / GPU / filtering 多模态查询时都需从 algorithm-style 系统（Faiss/DiskANN/SPANN）切换到 DBMS-style 系统（Milvus）。这是"维度质变"而非"规模质变"——可能在任何 N 上发生。

## Answer

### 四轴跨档质变表（新增 Milvus 行）

| 规模档 | 索引选择 | 存储介质 | 并行能力 | 工程形态（NEW） |
|---|---|---|---|---|
| < 10M | Flat brute force | DRAM 单机 | 无需 | library 即可 |
| 10M – 100M | NSG/HNSW (full) → IVF+HNSW coarse | DRAM | 单机多核 | library / 算法系统 |
| 100M – 1B | IVF+HNSW coarse + 压缩 (PQ/SQ) | DRAM 64 GB+ | 单机多核 | library / 算法系统 |
| **1B 跨档质变** | graph 全内存触顶；分支：(a) 全压缩 IVFPQ；(b) DiskANN graph+PQ+SSD；(c) SPANN IVF+SSD；(d) NSG 多机分片；**(e) Milvus shared-storage 分布式** | **DRAM → SSD** 或 **多机 DRAM** 或 **shared storage S3/HDFS** | 多 SSD 或多机内存或 K8s 弹性 | **DBMS 接管：Milvus = library 不能做的所有功能** |
| 1B – 100B | IVF1M_HNSW + PQ/SQ；SPANN/DiskANN/Milvus | SSD 必须；分布式 | 必须分布式 | DBMS 强烈推荐 |
| 100B → 1.5T（Meta 实测） | IVF + SQ6 + HNSW coarse + 分布式 mmap | mmap 分布式存储（83 TiB/20 服务器） | 中央 fan-out 网络瓶颈 | library 路线（Meta）vs DBMS 路线（无实测） |
| > 1.5T | wiki **未覆盖** | — | — | — |

### 四个明确的"质变点"（updated）

**质变点 1：~10M → 1M+ 量级时**（[per topics/index-selection.md] Fig 10 决策树阈值）
- Flat brute force → 真正需要索引

**质变点 2：1B 单机 RAM 触顶**
- HNSW / NSG 全图 1B+ OOM
- 三条算法路线（DiskANN / SPANN / NSG 分片）+ **新增 Milvus DBMS 路线**
- Recall 上限分化：
  - Faiss IVFPQ 全内存：~62%
  - DiskANN: 98.68%
  - SPANN: >90% @ ~1ms
  - **Milvus IVF_FLAT @ SIFT1B 单节点 1.5TB RAM**：throughput 高但 recall 数字论文未拆 [per benchmarks/milvus-vs-prior-sift10m-deep10m.md]

**质变点 3：百亿+ → 万亿**
- 单机 SSD 容量触顶
- 必须分布式 mmap + 极致压缩 [per benchmarks/faiss-trillion-scale.md]
- Meta 1.5T 实测延迟 ~1s——延迟语义本身质变

**质变点 4：从 algorithm-style 到 DBMS-style 的形态质变**（NEW，本次 ingest 显式暴露）
[per systems/milvus.md, wang-2021-milvus Table 1]：

algorithmic 系统（Faiss / DiskANN / SPANN）共同假设：
- 数据静态（一次构建后只读）
- 单租户（无 query optimizer / transaction / 隔离）
- 单维度查询（仅 vector similarity，无 filter / multi-vector）

引入下面任一需求**即触发 DBMS 形态切换**：
- **持续写入** → LSM / segment 模型
- **分布式 + 弹性扩缩** → shared-storage / consistent hashing
- **Attribute filter / multi-vector query** → query optimizer + 5 策略 / fusion
- **多租户隔离** → schema / collection 抽象

**关键观察**：质变点 4 与规模 N **正交**——可能在 1M 数据上因为 dynamic 需求触发 DBMS 化，也可能在 1B+ 静态数据上仍走 algorithm-style（Bing 用 SPANN）。

> **wiki 解读**：质变点 4 是 **engineering dimension** 而非 **scale dimension**。pre-snapshot 把"质变"窄化为规模阈值；Milvus ingest 后 wiki 显示出工程维度的质变同样根本。

### 不算质变的（参数微调档）

- 同一 IVF 框架下 K_IVF 参数调
- HNSW 内 M 参数调
- 同一 PQ 配置下 m 参数调
- Milvus segment 大小 / closure replicas 等也属此

### 已知盲区（trigger 后续 ingest）

- **百亿与千亿之间是否还有质变点**：Milvus 论文实测到 SIFT1B / 12 节点；**Bing/Pinecone 千亿+ DBMS 形态实测**未覆盖
- **Pinecone pod-based 架构的质变形态**：与 Milvus shared-storage 路线对立的另一种 DBMS 设计，wiki 未覆盖
- **Read-heavy vs write-heavy 路线分化**：Milvus 论文 §5.3 明确说"single writer is sufficient since Milvus is read-heavy"——write-heavy 场景质变点未覆盖
- **Milvus 2.0+ cloud-native 重写**：1.x 论文是本次基础；2.0+ DataNode/QueryNode/log broker 分层是质变 4 的演化
- **Embedding 升级触发的质变**：是另一种"事件型质变"，[query: embedding-update-handling] 仍未覆盖

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [systems/faiss.md](../../systems/faiss.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
