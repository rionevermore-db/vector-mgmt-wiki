---
query-key: quantization-landscape
date: 2026-05-12
phase: post
ingest-context: distributedann-cited-frontier
wiki-pages-total: 83
cited-pages: [concepts/distributedann-cited-frontier.md, concepts/product-quantization.md]
cited-count: 2
---

# Post-snapshot (distributedann-cited-frontier): quantization-landscape

## TL;DR

AiSAQ 把 PQ vectors 也放 SSD (DRAM-free), LM-DiskANN 让 neighbor PQ codes per node 一同读. 两者**改变 PQ storage placement**, 不改 quantization algorithm. CXL-ANNS 通过 memory disaggregation 扩展可承载 dataset size, 不直接 quantize. 关键 NEW: quantization storage placement (DRAM vs SSD vs CXL memory pool) 是**第 4 axis** (algorithm + post-hoc vs training-time + single vs multi-vector + storage placement).

## Cited Pages

- [concepts/distributedann-cited-frontier.md](../../concepts/distributedann-cited-frontier.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
