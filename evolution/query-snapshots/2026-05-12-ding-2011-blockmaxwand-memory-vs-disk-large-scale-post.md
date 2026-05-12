---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: ding-2011-blockmaxwand
wiki-pages-total: 82
cited-pages: [concepts/blockmaxwand-bm25.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (ding-2011-blockmaxwand): memory-vs-disk-large-scale

## TL;DR

BlockMaxWAND 增强 sparse-side SSD-friendly 优势——block-level pruning skip 整 block, **减少 SSD page reads**. Posting list 已是 sequential read SSD-friendly, BMW 额外让 SSD reads 数量级减少. 关键 NEW: BMW 让 sparse-side disk philosophy (50 年成熟 inverted index) 在 SSD 时代仍是 production primary path——比 dense ANN random read SSD-stress 更优.

## Cited Pages

- [concepts/blockmaxwand-bm25.md](../../concepts/blockmaxwand-bm25.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
