---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md]
cited-count: 1
---

# Post-snapshot (modern-embedding-paradigms): scale-tier-shifts

## TL;DR

NEW indirect: embedding model 选型在 scale tier 上**反向** drive 质变.
- **10亿规模**: GTE-base 110M + 768d 足够, 全 RAM 单节点
- **百亿规模**: Gecko 256d 推荐 (storage 1/3), 仍 RAM tier
- **千亿规模**: NV-Embed 4096d storage explode (1500GB float32 → PQ-32 必需), DRAM+SSD hybrid 切换点
- **万亿规模**: 任何 embedding model 4096d 都 over-budget RAM, MRL 256d 或 DiskANN/SPANN 路径

关键 NEW: model SOTA 选 ≠ best production——4096d 大 model 在 100B+ vectors 时 storage cost 反胃, **production 反 SOTA-pursue** 反而选 Gecko-256d 或 MRL truncate 256d 的 BGE-M3.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
