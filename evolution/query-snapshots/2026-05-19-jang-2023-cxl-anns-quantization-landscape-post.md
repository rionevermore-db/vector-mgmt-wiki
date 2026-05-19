---
query-key: quantization-landscape
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: [systems/cxl-anns.md, concepts/product-quantization.md]
cited-count: 2
---

# Post-snapshot (jang-2023-cxl-anns): quantization-landscape

## TL;DR

**间接影响——提供"何时可以不量化"的反论据**。CXL-ANNS 不引入新量化方法，但其核心主张挑战 quantization 的必要性前提：billion-scale 用 PQ 压缩 embedding table 后**精度随压缩率显著下降**（论文实测 45.8% 数据缩减后够不到 0.9 recall@k，Yandex-D 仅 0.58）。CXL-ANNS 的答案是"有 CXL 解耦内存就不压缩，全精度全放内存池"。对 quantization landscape 的意义：PQ/OPQ/SQ/RaBitQ 的"压缩-精度-内存"三角，其"为什么需要压缩"的前提（单机内存有限）在 CXL 解耦内存场景下被削弱——量化从"billion-scale 必需"降级为"无 CXL 硬件时的 fallback"。不改 PQ/RaBitQ 本身的相对排序。

## Cited Pages

- [systems/cxl-anns.md](../../systems/cxl-anns.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
