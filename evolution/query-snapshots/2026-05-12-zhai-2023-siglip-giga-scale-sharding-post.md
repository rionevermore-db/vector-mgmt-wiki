---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: zhai-2023-siglip
wiki-pages-total: 84
cited-pages: [concepts/siglip.md]
cited-count: 1
---

# Post-snapshot (zhai-2023-siglip): giga-scale-sharding

## TL;DR

SigLIP 是 embedding model, sharding axis 不直接关联. Production giga-scale multimodal retrieval 用 SigLIP-encoded embedding + 走 wiki vendor 各自 architecture variant (Vespa SPANN / Turbopuffer SPFresh / DistributedANN / etc.). SigLIP 训练成本数量级降低 (4 TPUv4 × 2 days) 让 self-hosted multimodal embedding 在 production 更可行.

## Cited Pages

- [concepts/siglip.md](../../concepts/siglip.md)
