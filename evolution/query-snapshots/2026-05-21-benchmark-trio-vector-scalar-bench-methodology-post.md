---
query-key: vector-scalar-bench-methodology
date: 2026-05-21
phase: post
ingest-context: benchmark-trio
wiki-pages-total: 105
cited-pages: [benchmarks/vectordbbench.md, benchmarks/big-ann-benchmarks.md, topics/ann-benchmarking-methodology.md, topics/attribute-filtering.md, benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
cited-count: 5
---

# Post-snapshot (benchmark-trio): vector-scalar-bench-methodology

## TL;DR

**重大 NEW——从"只有策略 taxonomy"升级到"taxonomy + 三层 benchmark 锚点 + 防作弊方法论"**：

1. **公平横测 filter 现有两个标准化 source 锚点** [per benchmarks/vectordbbench.md, benchmarks/big-ann-benchmarks.md]：
   - **系统级**：VectorDBBench Filtering case（int-based + label-based，30+ vendor 同 region 同硬件横测）。
   - **竞赛级**：big-ann NeurIPS 2023 **Filter track**（YFCC 10M × 192d + tag 过滤，FAISS baseline 3,200 QPS，标准化 Azure D8lds v5 + 私有 query set）。

2. **方法论四要素现 source-backed** [per topics/ann-benchmarking-methodology.md]：固定 embedding model + 标明 A/B/C strategy（pre/post/hybrid）+ **selectivity / range sweep** + 防 vendor 调参不对称。

3. **"benchmarks lie" 三陷阱显式化**：调参努力不对称（[per benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md] Milvus 用 single-thread + 默认参数）/ 厂商自测偏向（VDBBench 由 Zilliz 维护,benchANT fork 交叉验证）/ leaderboard 数字易变不可冻结。

4. **抽象层选择本身是答案**：问"filter 算法谁强"用算法级 / 问"生产选哪个 DB 的 filter 扛不扛"用系统级 VDBBench（含持续插入下 filter QPS 退化）。

仍开放：**空间(geo) + 三模(vector+scalar+spatial)公平横测零 source**（三个 benchmark 均无空间 track）。

## Cited Pages

- [benchmarks/vectordbbench.md](../../benchmarks/vectordbbench.md)
- [benchmarks/big-ann-benchmarks.md](../../benchmarks/big-ann-benchmarks.md)
- [topics/ann-benchmarking-methodology.md](../../topics/ann-benchmarking-methodology.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)
