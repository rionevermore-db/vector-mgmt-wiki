---
query-key: multimodal-bench-methodology
date: 2026-05-29
phase: pre
ingest-context: lance-geo
wiki-pages-total: 107
cited-pages: [topics/multimodal-embedding-retrieval.md, topics/ann-benchmarking-methodology.md, systems/vespa.md]
cited-count: 3
---

# Pre-snapshot (lance-geo): multimodal-bench-methodology

## TL;DR

三模(vector+scalar+spatial)benchmark 方法论:**空间能力不对称 + benchmark 全空白**。

- **空间能力 inventory**:wiki 记 **原生 spatial index 仅 Vespa**(position dimension);其余 vendor 缺或只能 bbox-as-scalar 近似 [per topics/multimodal-embedding-retrieval.md]。
- **算法共识**:严重 fragmented(geohash / R-tree / bbox)。
- **benchmark**:三大 ANN benchmark 全无 spatial track,三模公平横测**零 source** [per topics/ann-benchmarking-methodology.md]。

→ 空间是"算法不成熟 + benchmark 不成熟"双缺;multimodal 是"算法成熟 + benchmark 不成熟"。

## Cited Pages

- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [topics/ann-benchmarking-methodology.md](../../topics/ann-benchmarking-methodology.md)
- [systems/vespa.md](../../systems/vespa.md)
