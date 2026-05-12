---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md]
cited-count: 1
---

# Post-snapshot (modern-embedding-paradigms): giga-scale-sharding

## TL;DR

间接 NEW: 千亿/万亿规模 16-node + 1TB RAM 部署时, embedding model 选型直接影响 storage/RAM 占用——**GTE-base (110M) 推理~ 1ms/query 但 dim 不一定 fit**, **Gecko 256d 压缩 storage to 1/3** (1.5T × 256 × 4 = 1.5TB vs 768d 4.5TB), **NV-Embed-v2 (7B Mistral) 推理~ 50ms/query 在 16-node 上是 bottleneck**. 关键 NEW: production 千亿规模 dense embedding 选型 trilemma——SOTA score (NV-Embed) vs compact dim (Gecko) vs cheap inference (GTE), 与 sharding 拓扑设计正交但 storage budget 强约束.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
