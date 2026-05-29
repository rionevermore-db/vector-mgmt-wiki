---
title: Lance Format（ML-native 列式 file + table format）
type: concept
sources: [lance-blogs-2026, lancedb-docs-2026-05]
related: [../systems/lancedb.md, product-quantization.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../systems/chroma.md, ../systems/pgvector.md]
created: 2026-05-29
updated: 2026-05-29
---

# Lance Format

**TL;DR**: Lance 是 [LanceDB](../systems/lancedb.md) 底层的 **ML-native 开源列式格式**——分两层:**file format**(列式存储容器)+ **table format**(CRUD 协议:版本/manifest/fragment/事务)。与 Parquet 的根本区别是**random-access-first**:Parquet 为 sequential scan + row group 优化,点查要解压整页;Lance **v2 弃 row group**、用 8MiB 大页 + per-column 独立 metadata + 可插拔 encoding,使**随机点查比 Parquet 快 ~2000×**(100M 行实测)同时 scan 不退化。两种结构编码(mini-block 小类型 / **full-zip 大向量/多模态**)让 embedding 这类大定宽值**无读放大随机取**。table 层的 **2D 布局(fragment × data file)**让"加一列 = 追加文件、不重写"——直接解 ML 的 schema evolution + wide-data 痛点。是 wiki 内**唯一以 OSS 列式格式作 primary storage** 的 vendor 基础(类比 Iceberg/Delta/Hudi 在 lakehouse 的地位,但定位 ML/AI)。[per sources/docs/lance-blogs-2026/lance-blogs-digest.md + lance-blogs-digest-tier2.md]

## 提出背景

[per the-case-for-random-access-i-o, designing-a-table-format-for-ml-workloads, lance-v2 blogs]

传统列式格式(Parquet/ORC)为 **OLAP sequential scan** 设计——row group + page 让大扫描高效,但:
- **点查代价高**:1000 个随机 int lookup = **71B instructions**(random)vs **1.34T**(全列扫)≈ **19× CPU**;还要解压整页才能取单值。
- **ML 同时需要两种访问**:feature engineering 阶段是 columnar scan(几千列分析),inference / vector search / upsert 阶段是 random point lookup(固定小列集,按 ID 取)。存两份副本成本不可接受。
- **wide-data 诅咒**:给 TPC-H 加一个 3KB embedding 列,数据从 0% wide → 99% wide——大定宽值(向量)成主流,Parquet 的 page-per-value + 巨 offset index = RAM 爆炸。

→ Lance 的设计目标:**单一格式同时服务 scan + random access + 多模态大值 + 频繁 schema evolution**。

## 关键性质

### 1. file format:弃 row group（v2,2024-04 Weston Pace）

[per lance-v2 blog]

- **"row groups have outlived their usefulness"**:太小→元数据过多;太大→RAM 爆。
- 列内 page **不必相邻**;writer 按 buffer 满独立 flush → 最大化 page + 最小 RAM。
- **8MiB 大页**服务 scan;random access 用 column metadata 里的 skip table / zone map(不绑 page 边界)。
- **true column projection**:读单列不需读其他列的 metadata。
- **encoding 作为 extension**:reader/writer 完全不感知 encoding(protobuf 描述)→ 加新 encoding 不改格式(避免 Parquet 跨实现碎片化)。

### 2. 两种结构编码（structural encoding）

[per columnar-file-readers-in-depth-structural-encoding blog]

| 编码 | 适用 | 机制 | 读放大 |
|---|---|---|---|
| **mini-block** | 小类型(int/double/string/bool) | 4-8KiB block + 2-byte repetition index | 容忍小放大换压缩率 |
| **full-zip** | **大多模态(embedding 3-6KiB / doc / image / video)** | zip transparently-compressed buffer + 每值前置 control word(repetition+definition) | **单值读无放大** |

→ **full-zip 是 vector embedding 的关键**:Parquet 对大定宽值只能 page-per-value + 巨 offset index(RAM 重);Lance full-zip 无 repetition index 即可随机取。

### 3. table format:2D 布局 + 无重写 schema evolution

[per designing-a-table-format-for-ml-workloads blog]

- **2D 存储布局**:行 →(纵向)fragment →(横向)data file。
- **加列不重写**:加新列 = 每个 fragment 追加一个 data file,**不重写已有列**——解 ML "为所有行算了个新 feature" 的核心痛点(Parquet/Iceberg 的 schema evolution 只对新行向前生效,补旧行要全量重写)。
- versioning / manifest / fragment / 事务语义(详见 [systems/lancedb.md §G 索引版本语义](../systems/lancedb.md))。

### 4. 性能数字

[per benchmarking-random-access-in-lance + lance-format-v2-2-benchmarks blogs]

- **随机点查**:100M 行(1000-char string),Lance **0.0006 s/key** vs Parquet **1.25 s/key** ≈ **~2000×**(作者称早先"100×"是低估)。
- **v2.2**:text-heavy 比 Parquet **小 52%**(FineWeb)/ blob random fetch **75×** / 加列 **61×**(13ms vs 520s)。
- **cloud 经济**:random access 在合适 page size 下可饱和 S3(~4K IOPS / 4GB/s);per 1000 query CPU 成本 random ~$0.0044 vs scan ~$0.012。

## 与同类对比

| | Lance | Parquet/ORC | Iceberg/Delta/Hudi(table format) |
|---|---|---|---|
| 优化目标 | **random + scan + 多模态** | sequential scan(OLAP) | analytical 数据交换标准 |
| row group | **无(v2)** | 有 | (依赖底层 file format) |
| 大向量值 | **full-zip 无放大随机取** | page-per-value + 巨 offset index | 不支持 native 多模态 |
| 加列 | **追加文件不重写** | 重写 | 仅新行向前(补旧行要重写) |
| 随机点查 | **~2000× Parquet** | baseline | 弱(manifest tree 瓶颈) |
| 定位 | **ML/AI** | 通用分析 | 企业数据湖 |

> **vs Iceberg(互补非竞争)** [per iceberg-and-lance blog]:Iceberg = 分析数据交换标准;Lance = ML/AI 格式。Iceberg 缺 native 多模态类型 + 低延迟 random access + 廉价 schema evolution。**Iceberg 的 pluggable DataFile reader/writer API 让 Lance 数据可通过 Iceberg 接口查**——两者可共存(详见 [systems/lancedb.md](../systems/lancedb.md))。

## 典型实现

Lance format 是 [LanceDB](../systems/lancedb.md) 的存储底座;也可独立用(Python `lance` / Rust)。向量索引(IVF*/HNSW/RaBitQ)+ R-Tree 空间索引 + scalar(BTREE/BITMAP/FTS)都是建在 Lance format 上的 secondary index。对接 Arrow 生态(GeoArrow 扩展类型 white-box)+ DataFusion(查询)+ HuggingFace(数据集分发)。

## Open Questions

- **full-zip 对 1B+ scale 的 random access 实测**:blog 实测到 100M 行;十亿级 random point lookup 的 IOPS 上限(对接 [systems/lancedb.md §I 10B 分布式])未单独量化。
- **encoding extension 的版本兼容**:protobuf-described encoding 跨 reader 版本的前后兼容保证未深入。
- **vs DiskANN/SPANN 的 SSD layout 对比**:Lance full-zip + 8MiB page 与 [DiskANN](../systems/diskann.md) 的 4KB-aligned block / [Starling block shuffling](./block-shuffling.md) 是不同 disk layout 哲学——head-to-head 未对比。
- **Lance-Iceberg 集成成熟度**:DataFile API 互通是 emerging,production 案例 wiki 未覆盖。
