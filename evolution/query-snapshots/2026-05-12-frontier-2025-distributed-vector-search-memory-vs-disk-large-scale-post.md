---
query-key: memory-vs-disk-large-scale
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md]
cited-count: 1
---

# Post-snapshot (frontier-2025-distributed-vector-search): memory-vs-disk-large-scale

## TL;DR

**NEW** (SPIRE 直接答案): SPIRE = 当前最清晰 OSS 学术 hierarchical SSD 系统配方——**top level (proximity graph) in-memory + 下层 vector clusters 全在 SSD**, bottom-up 构造直到 root fit single server memory. 关键 quantitative NEW:
1. SPIRE 8B vectors / 46 nodes = ~200M vec/node, **stateless compute tier** (in-memory root replicated, SSD level 可 reconstruct)——千亿 elastic scaling 关键
2. SPIRE saturate SSD I/O 时仅 <30% network + <40% CPU——**bottleneck 真正在 SSD I/O**, 不是 network/compute
3. SPIRE 比 DSPANN / Pinecone-pod / SPTAG / Manu 更系统化 end-to-end accuracy 优化
4. wiki disk philosophy 现 10 类 (CXL / DRAM-free / SSD hybrid / ...) + SPIRE = 11 类: **multi-level SSD with end-to-end accuracy budget**——独立设计 axis

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
