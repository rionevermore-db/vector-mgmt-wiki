---
query-key: multimodal-bench-methodology
date: 2026-05-21
phase: post
ingest-context: benchmark-trio
wiki-pages-total: 105
cited-pages: [benchmarks/big-ann-benchmarks.md, topics/ann-benchmarking-methodology.md, topics/multimodal-embedding-retrieval.md]
cited-count: 3
---

# Post-snapshot (benchmark-trio): multimodal-bench-methodology

## TL;DR

**中等 NEW——cross-modal 维度首次有 algorithm-level benchmark 锚点,但三模仍空**：

1. **cross-modal benchmark 不再完全空白** [per benchmarks/big-ann-benchmarks.md]：big-ann NeurIPS 2023 **OOD track** = Yandex Text-to-Image 10M × 200d，**query 与 base 分布不同**——正是 cross-modal（文搜图）核心挑战,DiskANN baseline 4,882 QPS @ 标准化 Azure 硬件 + 私有 query set。这是 wiki 内**首个 source-backed cross-modal ANN 公平评测**。

2. **但仍是"算法级 + 单数据集"**：big-ann OOD 测的是索引在分布偏移下的 recall-QPS,**不是 vendor-level multimodal head-to-head**——后者依旧零 source（呼应 [per topics/multimodal-embedding-retrieval.md] "没有 vendor 公开 multimodal head-to-head benchmark"）。

3. **空间 / 三模(vector+scalar+spatial)横测依旧零 source** [per topics/ann-benchmarking-methodology.md Open Questions]：三个 benchmark 全无空间 track;能力严重不对称（真索引 / bbox 近似 / 完全没有）的公平处理方法仍无答案。

4. **OOD 作为"分布偏移"benchmark 范式可迁移**：cross-modal 的本质是 query/base 分布不同,这个口径理论上可推广到"老 embedding 索引 + 新 embedding query"的跨模型迁移评测——但 source 未做此连接（推测）。

净结论：**2 模(cross-modal)有锚点,3 模(+spatial)仍是 wiki 最硬的 benchmark 盲点**。

## Cited Pages

- [benchmarks/big-ann-benchmarks.md](../../benchmarks/big-ann-benchmarks.md)
- [topics/ann-benchmarking-methodology.md](../../topics/ann-benchmarking-methodology.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
