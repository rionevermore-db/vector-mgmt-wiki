---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: [concepts/splade-family-baselines.md, concepts/splade-sparse-retrieval.md]
cited-count: 2
---

# Post-snapshot (splade-family-baselines): giga-scale-sharding

## TL;DR

无直接影响——sparse retrieval evolution chain (doc2query → DeepImpact → COIL → SPLADE) 是 IR algorithm 维度，与千亿/万亿物理分片正交. 但 **indirect**: 千亿 hybrid retrieval 若选 sparse+dense fusion，sparse 一侧基于 BMW-style block pruning posting list + neural weights (SPLADE/DeepImpact)，shard 内 inverted index 维度仍是 vocabulary-partitioned 而非 vector-partitioned，**两套 sharding 拓扑共存**是 production hybrid 隐性约束.

## Cited Pages

- [concepts/splade-family-baselines.md](../../concepts/splade-family-baselines.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
