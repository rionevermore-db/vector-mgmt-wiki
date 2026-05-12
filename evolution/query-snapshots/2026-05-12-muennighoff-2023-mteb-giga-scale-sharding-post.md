---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md]
cited-count: 1
---

# Post-snapshot (muennighoff-2023-mteb): giga-scale-sharding

## TL;DR

MTEB 不直接评估千亿规模 sharding (是 embedding model benchmark 而非 system benchmark). 但 MTEB-validated embedding model 是 production 千亿 deployment 的 embedding source——sharding 策略决策 + MTEB embedding model 选择 是**两个独立 axis**, 但**联合优化** in production. 关键 NEW: 千亿规模选 embedding model 必须考虑 (a) MTEB quality, (b) embedding dim (impacts storage cost — MRL prefix-aware embedding model 优势), (c) inference cost.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
