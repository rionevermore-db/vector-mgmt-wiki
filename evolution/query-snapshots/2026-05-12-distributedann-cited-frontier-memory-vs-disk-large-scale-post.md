---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: distributedann-cited-frontier
wiki-pages-total: 83
cited-pages: [concepts/distributedann-cited-frontier.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (distributedann-cited-frontier): memory-vs-disk-large-scale

## TL;DR

4 paper 全部围绕 memory-vs-disk trade-off 创新. AiSAQ + LM-DiskANN 极端 DRAM 减少 (~10 MB billion-scale); CXL-ANNS 通过 CXL memory pool 扩展 DRAM. 关键 NEW: wiki 8 类 disk philosophy 之外, AiSAQ / LM-DiskANN 添加**第 9 类: DRAM-free 单节点 disk-resident 完整 ANN**; CXL-ANNS 添加**第 10 类: CXL memory disaggregated ANN**. 总 10 类 disk philosophy 完整 ANN deployment landscape.

## Cited Pages

- [concepts/distributedann-cited-frontier.md](../../concepts/distributedann-cited-frontier.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
