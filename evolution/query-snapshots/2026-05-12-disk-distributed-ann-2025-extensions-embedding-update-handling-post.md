---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: []
cited-count: 0
---

# Post-snapshot (disk-distributed-ann-2025-extensions): embedding-update-handling

## TL;DR

无直接影响——3 paper 都假设 embedding 固定. 间接: SPI 的多分辨率 hierarchical chunk index 在 embedding model 升级时 cost **更高** (每个 resolution level 都需重 encode + 重 build), 是 SPI 部署劣势.

## Cited Pages

(无)
