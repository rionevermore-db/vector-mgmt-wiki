---
query-key: scale-tier-shifts
date: 2026-05-12
phase: post
ingest-context: distributedann-cited-frontier
wiki-pages-total: 83
cited-pages: [concepts/distributedann-cited-frontier.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (distributedann-cited-frontier): scale-tier-shifts

## TL;DR

关键 NEW: **3 类 ANN hardware deployment paradigm identified**——(1) DRAM-resident (HNSW classic), (2) DRAM + SSD hybrid (DiskANN/SPANN/Starling), (3) **DRAM-free / Memory disaggregated** (AiSAQ / LM-DiskANN / CXL-ANNS), (4) Distributed shared (DistributedANN). 4 paper 引入 paradigm 3, 让 tier-shift 不仅是 scale 维度, 也是 hardware tier 维度. CXL hardware 成熟度 + AiSAQ multi-dataset switching production case 是 future tier 5 (>1T) hardware path candidate.

## Cited Pages

- [concepts/distributedann-cited-frontier.md](../../concepts/distributedann-cited-frontier.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
