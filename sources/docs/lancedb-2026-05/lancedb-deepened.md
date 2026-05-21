# LanceDB docs deepened — captured content (WebFetch, 2026-05-21)

> 来源：docs.lancedb.com(indexing / quantization / storage / enterprise / geneva 深页)。Layer-1 citation 锚点。

## Enterprise 架构(3-plane disaggregation)

[enterprise/architecture.md]

- **Data Plane**:query nodes(client-facing:接收请求 + 校验 + 解析目标 table + plan + 返回结果)+ **plan executors**(query node 后面的 read-execution 层,对 object storage 做 cache-backed reads)+ **indexers**(后台重活:build index / merge index state / compaction)。
- **Control Plane**:configuration / service discovery / identity integration / policy / cluster lifecycle。
- **Object Storage**:table data + manifests + index artifacts 全在这里——"durable record of the table lives outside any single query node"。
- **核心设计**:request handling / read execution / index-building **三者独立 scale**,不抢同一份 compute。"Query fleets can scale for interactive traffic without also scaling background indexing capacity."
- 一致性模型、多级缓存细节:本页未明示(仅提 plan executor 的 cache-backed reads)。

## Enterprise benchmark(vendor 自测)

[enterprise/benchmarks.md] —— **LanceDB 自家发布,self-published**

- 数据集:dbpedia-entities-openai-1M(**1M × 1536d**)+ synthetic(**15M × 256d**)
- Vector search(warmed cache):**P50 25ms / P90 26ms / P99 35ms / max 49ms**
- Full-text search:P50 26ms / P90 37ms / P99 42ms / max 98ms
- Vector + 选择性 metadata filter(如 "Sci-fi 2000-2012"):P50 30ms / P90 39ms / P99 50ms
- Vector + 宽 metadata filter(如 "Sci-fi since 1900"):P50 65ms / P90 76ms / **P99 100ms**
- "thousands of QPS in some deployments"(无具体吞吐数)
- **无 recall / 无 ingestion rate / 无硬件规格 / 无对比系统**

## 向量索引族

[indexing/vector-index.md]

索引类型(比 2026-05-12 抓到的 "IVF + HNSW" 丰富得多):
- **IVF_FLAT**:raw vector,无量化损失
- **IVF_PQ**:dim ≤256 时 accuracy 常优于 IVF_RQ
- **IVF_RQ**:**RaBitQ-style quantization**,极强压缩
- **IVF_SQ**:scalar quantization
- **IVF_HNSW_FLAT**:最高 recall 无量化
- **IVF_HNSW_SQ**:**best recall/latency trade-off**
- **IVF_HNSW_PQ**:IVF partition + HNSW graph
- binary 向量:仅 **IVF_FLAT + hamming**

Build 参数:
- HNSW-backed:`num_partitions` 起点 `num_rows // 1_048_576`;`ef_construction` 起点 150
- IVF_RQ / IVF_PQ:`num_partitions` 起点 `num_rows // 4096`
- IVF_PQ:`num_sub_vectors` 起点 `dimension // 8`
- HNSW:`m`(每点邻居数)、`ef_construction`

距离:l2(默认)/ cosine / dot / hamming(仅 binary)。Multivector indexing 当前要求 `distance_type="cosine"`。

Search 参数:`limit`(=k)、`nprobes`(默认 auto-tune)、`minimum_nprobes`(总是扫)/`maximum_nprobes`(上界)、`ef`(起点 1.5×k,上到 10×k)、`refine_factor`(多读候选 + 内存重排)。filter 激活时先扫 minimum_nprobes,不够 limit 再扩到 maximum_nprobes。

**GPU index building:本页(indexing + quantization)未提及。** —— 与 2026-05-12 partial ingest "GPU support for vector index building" 的 claim **不相互印证**(needs verification)。

## Quantization

[indexing/quantization.md]

4 种量化索引:
- **IVF_PQ**(默认):IVF + Product Quantization
- **IVF_RQ**(RaBitQ):**1 bit/dim** binary quantization;`num_bits` 默认 1(更高 fidelity 更高存储);`max_iterations` 默认 50;`sample_rate` 默认 256 samples/partition。**1024-d float32 4KB → ~几百 bytes**
- **IVF_HNSW_SQ**:IVF + per-partition HNSW + scalar quantization
- **IVF_HNSW_PQ**:IVF + per-partition HNSW + product quantization

两个正交轴:partition 搜索策略(flat IVF vs HNSW graph)× 压缩方法(PQ / RQ / SQ)。默认推荐 IVF_PQ;极端压缩用 IVF_RQ;最佳 recall/latency 用 HNSW-backed。

## Storage(5-tier latency 模型)

[storage/index.md]

- **immutable fragments** 是存储原语 → "separates storage and compute and writes immutable fragments, making it a strong fit for stateless, horizontally scalable deployments"
- 5 层后端(高延迟→低延迟):
  1. Object Storage(S3/GCS/Azure)——hundreds of ms,**effectively unlimited 但 QPS bound by concurrency limits**
  2. File Storage(EFS/GCS Filestore)——p95 <~100ms
  3. Third-party(MinIO/WekaFS)——<100ms
  4. Block(EBS/GCP)——often <30ms,**not shareable across instances**
  5. Local(SSD/NVMe)——p95 often <10ms,not shareable
- 多版本:opening table with many versions 在 object store 上"dominated by listing cost"(`new_table_enable_v2_manifest_paths`)

## Geneva(多模态 feature engineering,Enterprise-only)

[geneva/index.md]

- 用途:把 raw data 转成 ML feature——"multimodal feature engineering at scale"
- 机制:Python **UDF 作为 virtual column**——prototype 函数 → UDF decorator 包装 → `Table.add_columns()` 注册为 virtual column → 配置执行环境 → backfill
- 执行:本地 / **Ray cluster** / **Kubernetes(KubeRay)**
- 与 Lance table 一体:feature 是 Lance dataset 的 virtual column,feature 计算内嵌进存储/检索而非独立 pipeline
- 解决:prototype → production 的依赖/版本管理,API `geneva.connect()` / `Table.add_columns()` / backfill
