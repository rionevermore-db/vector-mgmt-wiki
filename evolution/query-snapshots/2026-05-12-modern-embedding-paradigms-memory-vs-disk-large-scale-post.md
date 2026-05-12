---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md]
cited-count: 1
---

# Post-snapshot (modern-embedding-paradigms): memory-vs-disk-large-scale

## TL;DR

间接 NEW: embedding model 输出 dim 决定 memory budget. **Gecko 256-dim 用 1/3 memory** vs 768-dim—— 给定 1.5T vectors, 256d float32 = 1.5TB (16-node 内可全 RAM), 768d float32 = 4.5TB (需 disk). 即 embedding model 选型直接影响 memory-vs-disk 决策边界, 256d Gecko/MRL 把 disk 切换点延后. NV-Embed-v2 4096-dim 反向放大问题——SOTA accuracy 强制 disk path.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
