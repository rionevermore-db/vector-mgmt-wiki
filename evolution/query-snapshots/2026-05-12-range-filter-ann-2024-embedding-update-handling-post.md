---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: range-filter-ann-2024
wiki-pages-total: 97
cited-pages: []
cited-count: 0
---

# Post-snapshot (range-filter-ann-2024): embedding-update-handling

## TL;DR

无影响——3 paper 都假设 embedding 固定. 间接: UNIFY 支持 incremental update 是稀有特性——embedding model 升级 + corpus 渐进迁移场景下, **UNIFY 比 SeRF 优 (后者 不支持 incremental)** 是 production cross-model migration 的 algorithm-level enabler.

## Cited Pages

(无)
