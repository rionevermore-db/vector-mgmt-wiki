---
query-key: multimodal-bench-methodology
date: 2026-05-29
phase: post
ingest-context: lance-geo
wiki-pages-total: 107
cited-pages: [systems/lancedb.md, topics/multimodal-embedding-retrieval.md, topics/ann-benchmarking-methodology.md, systems/vespa.md]
cited-count: 4
---

# Post-snapshot (lance-geo): multimodal-bench-methodology

## TL;DR

**空间能力 inventory 更新——但 benchmark 缺口纹丝不动,反而把"capability ≠ benchmark"讲得更清楚**:

1. **原生 spatial index 现 2 家**(此前仅 Vespa):[LanceDB](../../systems/lancedb.md) 2026-02 加 **R-Tree**(static/immutable 2D bbox + Hilbert packed-build + GeoArrow 类型 + GeoDataFusion ST_* 函数)[per systems/lancedb.md §H, sources/docs/lance-geo-2026-02/geo-support.md]。比 Vespa 的 R-tree-like 更明确。
2. **算法共识仍 fragmented**(geohash / R-tree / bbox)——2 家用的也不是同一套(Vespa position vs LanceDB R-Tree)。
3. **benchmark 依旧零**:三大 ANN benchmark 仍无 spatial track,三模(vector+scalar+spatial)公平横测**无 source**——capability 增长**没有**带来 benchmark。

→ **关键洞察固化**:空间从"算法 + benchmark 双缺"变成"**算法开始成熟(2 vendor 原生)、benchmark 仍空白**"——这正是 wiki "algorithm-成熟先于 benchmark-成熟" 的活案例,也说明 **vendor 加 capability ≠ 社区有公平横测**。三模 benchmark 仍是 wiki 最硬的真空。

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)（§H 原生 R-Tree spatial）
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [topics/ann-benchmarking-methodology.md](../../topics/ann-benchmarking-methodology.md)
- [systems/vespa.md](../../systems/vespa.md)
