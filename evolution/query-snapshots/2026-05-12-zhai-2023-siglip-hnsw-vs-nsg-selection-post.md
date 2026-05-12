---
query-key: hnsw-vs-nsg-selection
date: 2026-05-12
phase: post
ingest-context: zhai-2023-siglip
wiki-pages-total: 84
cited-pages: [concepts/siglip.md, concepts/clip.md]
cited-count: 2
---

# Post-snapshot (zhai-2023-siglip): hnsw-vs-nsg-selection

## TL;DR

SigLIP 是 embedding model 不是 ANN. SigLIP-encoded vectors 与 CLIP-encoded vectors 都 L2-normalized + cosine ANN-friendly, 走相同 HNSW path. HNSW production OSS deployment 仍 7 vendor, ANN choice 不变.

## Cited Pages

- [concepts/siglip.md](../../concepts/siglip.md)
- [concepts/clip.md](../../concepts/clip.md)
