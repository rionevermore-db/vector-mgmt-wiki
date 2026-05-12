---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: []
cited-count: 0
---

# Post-snapshot (splade-family-baselines): scale-tier-shifts

## TL;DR

无影响——sparse retrieval evolution chain 在 10 亿/百亿/千亿/万亿 各档都用同一套 inverted index + BMW 机制，**不存在 sparse 维度的质变档**. 关键 NEW: sparse retrieval 是 scale-invariant baseline——任何规模 dense ANN 出现 recall 跌穿时，sparse 这边稳定无质变，**作为 fallback rerank 候选始终可用**.

## Cited Pages

(无)
