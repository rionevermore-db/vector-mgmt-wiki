# Lance/LanceDB 技术 blog digest — Tier-2 (WebFetch, 2026-05-29)

> 来源:lancedb.com/blog 格式内核 + 对比篇。Layer-1 citation 锚点。Tier-1 在 lance-blogs-digest.md。

## the-case-for-random-access-i-o（格式设计 rationale）

- Parquet row group 为 sequential scan 优化,点查要解压整页:1000 random int lookup = **71B instructions** vs 全扫 **1.34T**(≈19× CPU)。
- ML 同时需 scan(feature eng)+ random(inference/vector search/upsert by ID);存两份不可接受。
- Lance 弃 row group,用 **8MiB 大页** + coalesced I/O;packed struct encoding(行主存常共取字段)。
- cloud:S3 ~4K IOPS / 4GB/s,合适 page size 可饱和;per 1000 query random CPU ~$0.0044 vs scan ~$0.012。

## designing-a-table-format-for-ml-workloads（file vs table format）

- file format = 列式容器;table format = CRUD 协议(versioning / manifest / fragment / 事务)。
- **2D 布局**:行→(纵)fragment→(横)data file。
- **加列不重写**:加新列 = 每 fragment 追加 data file,不重写已有列(解 ML "为所有行算新 feature");Parquet/Iceberg schema evolution 仅新行向前。
- wide-data 诅咒:加 3KB embedding 列 → TPC-H 0%→99% wide。

## columnar-file-readers-in-depth-structural-encoding

- structural encoding = 把 compressed buffer 写进 disk page 的方式。
- **mini-block**(int/double/string/bool):4-8KiB block + 2-byte repetition index,容忍小放大换压缩。
- **full-zip**(embedding 3-6KiB / doc / image / video):zip transparently-compressed buffer + 每值 control word(repetition+definition)→ **单值读无放大**;Parquet 对应是 page-per-value + 巨 offset index(RAM 重)。

## benchmarking-random-access-in-lance

- 100M 行(1000-char string),Intel E-2176M / 128GiB / 980 PRO 2TB:Lance **0.0006225 s/key** vs Parquet **1.2467 s/key** ≈ **~2000×**(作者称早先 "100×" 低估)。

## lance-v2（2024-04-13, Weston Pace）

- 弃 row group("clever idea outlived usefulness"):size dilemma + 多线程低效(10 核读 50 IOPS 同步等)。
- 列内 page 不必相邻,writer 独立 flush → 最大化 page + 最小 RAM。
- **encoding 作 extension**(protobuf,reader/writer 不感知)→ 加 encoding 不改格式。
- true column projection:读单列不读其他列 metadata。scan(8MiB page)+ random(column metadata skip table/zone map)兼顾。

## opensearch-vs-lancedb（⚠️ LanceDB 自测,Justin Miller @ LanceDB）

- COCO 2017(287,360 图,SigLIP-2 **1152-d** L2-norm,~160KB/图),projected 1M/10M/100M。
- 287K:both sub-50ms p95 top-10;recall@10 both >0.95;LanceDB index build 68s/287K。
- **成本 @ 100M**:OpenSearch r6g.12xlarge(384GB)~**$3,333/mo** vs LanceDB c6g.4xlarge(32GB)~**$779/mo** = **4.3× cheaper**;"OpenSearch scales with index RAM,LanceDB scales with QPS not corpus size"。
- OpenSearch 赢:feature breadth(FTS/BM25/filter/agg 同 query)、security/multi-tenancy、sub-10ms p99 无需调 cache。
- **vendor-authored**——按 benchmarks-lie 第 2 陷阱处理。

## the-future-of-open-source-table-formats-iceberg-and-lance

- 互补非竞争:Iceberg = 分析数据交换标准;Lance = ML/AI 格式。
- Iceberg 缺:native 多模态类型、低延迟 random access、廉价 schema evolution(manifest tree 瓶颈)。
- **Iceberg pluggable DataFile reader/writer API → Lance 数据可经 Iceberg 接口查**(emerging 互通)。
