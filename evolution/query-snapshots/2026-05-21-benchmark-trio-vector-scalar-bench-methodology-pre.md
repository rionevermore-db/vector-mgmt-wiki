---
query-key: vector-scalar-bench-methodology
date: 2026-05-21
phase: pre
ingest-context: benchmark-trio
wiki-pages-total: 101
cited-pages: [topics/attribute-filtering.md, concepts/range-filter-ann-2024.md, benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
cited-count: 3
---

# Pre-snapshot (benchmark-trio): vector-scalar-bench-methodology

## TL;DR

当前 wiki 能给出 **filter-aware ANN benchmark 的方法论骨架**,但**缺一个 dedicated benchmark-tooling / system-level harness page**——只有"按论文各自实验"散落的 per-paper benchmark page。

能答的:
- **5 策略框架** [per topics/attribute-filtering.md]:pre-filter / post-filter / hybrid，Milvus partition-based (E) 比 cost-based (D) 快 13.7×。
- **A/B/C taxonomy + range size sweep** [per concepts/range-filter-ann-2024.md]:fair bench 必须扫 0.1%/1%/10%/50%/100% range，并标明 vendor 用哪种 strategy。
- **单点 head-to-head 例子** [per benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]:ACORN vs FilteredVamana/NHQ/Milvus，但 Milvus 用 single-thread + 默认参数 → 论文自承可能低估 production。

答不出的(当前盲点):
- **没有跨厂商标准化 benchmark 工具**的方法论(怎么公平测 30+ vector DB 的 filter 性能、统一硬件、QP$ 成本)。
- **没有 system-level 运维指标**(load duration / 持续插入下的 filter QPS / streaming insertion-under-load)。
- benchmark "可信度" 维度(timeout、geometric-mean 打分、是否 vendor 自测偏向)无 source 支撑。

## Cited Pages

- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [concepts/range-filter-ann-2024.md](../../concepts/range-filter-ann-2024.md)
- [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)
