---
query-key: giga-scale-sharding
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: [systems/cxl-anns.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (jang-2023-cxl-anns): giga-scale-sharding

## TL;DR

**间接影响——新增一个"如果有 CXL 硬件"的替代分片哲学**。本 query 锁定 16 节点私有云（128U+1TB RAM）的千亿/万亿分片。CXL-ANNS 不直接答这个（它是单 cluster CXL 内存池，非 16 独立节点 share-nothing 分片，且无 vendor production，仅 FPGA+gem5），但提供一个 framing 反论：**若内存通过 CXL 解耦池扩容（≤4 PB），billion-scale 可不分片不压缩——单一全精度图 + near-data 算距离**。对私有云部署的启示：传统答案（千亿全内存 IVF / 万亿 mmap+极端压缩）假设单机内存固定；CXL 解耦把"内存"从节点本地资源变成 cluster 内可组合 pool，改变分片必要性的前提——但 CXL-ANNS multi-host 仅扩到 4 host（6 host PE 瓶颈），尚不覆盖 16 节点规模的实证。仍属 wiki 内 zero production example for 1T@50ms。

## Cited Pages

- [systems/cxl-anns.md](../../systems/cxl-anns.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
