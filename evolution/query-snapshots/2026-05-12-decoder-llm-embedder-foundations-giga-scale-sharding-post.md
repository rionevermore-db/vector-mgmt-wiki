---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: decoder-llm-embedder-foundations
wiki-pages-total: 89
cited-pages: [concepts/decoder-llm-embedder-foundations.md]
cited-count: 1
---

# Post-snapshot (decoder-llm-embedder-foundations): giga-scale-sharding

## TL;DR

无直接影响——三 paper 都是 model training 侧而非 sharding 侧. 间接 NEW: GritLM 的"生成 + 嵌入 单 model 双模式"提示**千亿规模 production stack 可减少一个 GPU 池**——如果选 GritLM-style unification, embedding-only 池可省, 但 vector search 仍需 disaggregation (Trinity 模式). 实际选择取决于 RAG batch heterogeneity.

## Cited Pages

- [concepts/decoder-llm-embedder-foundations.md](../../concepts/decoder-llm-embedder-foundations.md)
