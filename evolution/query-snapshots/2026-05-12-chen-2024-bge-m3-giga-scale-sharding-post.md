---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, systems/vespa.md, systems/milvus.md]
cited-count: 3
---

# Post-snapshot (chen-2024-bge-m3): giga-scale-sharding

## TL;DR

BGE-M3 千亿规模 production case 不公开. 关键 NEW: BGE-M3 输出 3 representation 占 storage = dense (1024-d × 4 bytes = 4 KB) + sparse (~50 non-zero × 8 bytes = 400 bytes) + colbert (128 token × 20 bytes = 2.5 KB) ≈ **6.9 KB/doc** total. 千亿 docs × 6.9 KB = ~690 TB——比 single dense 768-d ≈ 300 TB 多 2.3×, 但**集成 3-way hybrid** quality 更高.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
