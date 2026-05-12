---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, topics/multimodal-embedding-retrieval.md]
cited-count: 2
---

# Post-snapshot (chen-2024-bge-m3): multimodal-bench-methodology

## TL;DR

BGE-M3 text-only (vs CLIP / SigLIP cross-modal). 关键 NEW: BGE-M3 + cross-modal embedding (CLIP) hybrid 是 production multimodal pipeline 一类——BGE-M3 处理 text-side retrieval (3-way), CLIP 处理 image-side retrieval. Vendor 端需要同时支持两 model 各自 output——Vespa multi-tensor field + tensor framework 是唯一 full-native.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
