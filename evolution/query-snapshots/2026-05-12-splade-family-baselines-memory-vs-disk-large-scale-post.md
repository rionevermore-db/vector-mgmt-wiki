---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: []
cited-count: 0
---

# Post-snapshot (splade-family-baselines): memory-vs-disk-large-scale

## TL;DR

无影响——sparse retrieval inverted index 本身就是 disk-friendly (posting list 可 mmap)，但本次 ingest 没 introduce new disk-tier strategy. SPLADE / DeepImpact / COIL 都假设 inverted index 已落地 (Lucene/Tantivy)，是 algorithm 层 advancement 而非 storage tier.

## Cited Pages

(无)
