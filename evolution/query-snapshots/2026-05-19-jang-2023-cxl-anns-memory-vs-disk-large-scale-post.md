---
query-key: memory-vs-disk-large-scale
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: [systems/cxl-anns.md, topics/disk-vs-memory-ann.md, systems/diskann.md, systems/spann.md, systems/distributedann.md, concepts/product-quantization.md]
cited-count: 6
---

# Post-snapshot (jang-2023-cxl-anns): memory-vs-disk-large-scale

## TL;DR

**直接且高影响**——CXL-ANNS 引入 billion-scale 的**第三条 scale 轴**。此前答案只有两条：(1) 量化压缩+全内存（Faiss IVFPQ，recall 卡 60-70%）；(2) hierarchical SSD/PMEM（DiskANN/SPANN/HM-ANN，storage 占 query latency 87.6%，比无限-DRAM oracle 差 29.4×/64.6×）。现在第三条：**CXL 解耦内存池**——全量 graph+embedding 不压缩不下放，用 caching+prefetch+near-data 算距离藏掉 far-memory（naive 比 oracle 慢 3.9×），最终比 oracle 还快 3.8× throughput / 低 68% latency，比 SOTA 111.1× QPS。**唯一同时做到 billion-scale + 全精度无损 + 低延迟**——代价是需 CXL 2.0+ 硬件（论文仅 FPGA+gem5，无 vendor production）。瓶颈从"存储容量"转移到"EP-side PE 算力"，scale-out 是加 EP/host 而非加 SSD。disk-vs-memory-ann.md 新增"关键洞见 6"+ memory hierarchy 表加 CXL 行。

## Cited Pages

- [systems/cxl-anns.md](../../systems/cxl-anns.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [systems/diskann.md](../../systems/diskann.md) / [systems/spann.md](../../systems/spann.md)（论文 hierarchical baseline）
- [systems/distributedann.md](../../systems/distributedann.md)（同 near-data computation, 不同 scale 路径）
- [concepts/product-quantization.md](../../concepts/product-quantization.md)（论文 compression baseline，45.8% 缩减后够不到 0.9 recall）
