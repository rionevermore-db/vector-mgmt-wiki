---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: zhai-2023-siglip
wiki-pages-total: 84
cited-pages: [concepts/siglip.md, concepts/clip.md]
cited-count: 2
---

# Post-snapshot (zhai-2023-siglip): embedding-update-handling

## TL;DR

SigLIP 已成为 multimodal embedding production default替代 CLIP. 关键 NEW: 客户从 CLIP-encoded corpus 升级到 SigLIP 不能 algorithm-level 复用 (CLIP 与 SigLIP cosine space 不互相 compatible)——需重 embed + 重建 ANN index. Talk SIGMOD 2026 live demo cross-model preserve frontier 在 SigLIP context 同样未关闭.

## Cited Pages

- [concepts/siglip.md](../../concepts/siglip.md)
- [concepts/clip.md](../../concepts/clip.md)
