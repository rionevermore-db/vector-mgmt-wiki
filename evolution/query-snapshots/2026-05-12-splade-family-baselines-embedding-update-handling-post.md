---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: [concepts/splade-family-baselines.md]
cited-count: 1
---

# Post-snapshot (splade-family-baselines): embedding-update-handling

## TL;DR

间接 NEW**: doc2query (Nogueira & Lin 2019) 是 "**doc 侧增强而非 query/embedding 侧增强**" 的代表——T5 生成的 expansion queries 写回 doc 作为 inverted index payload，**embedding model 升级时只需重跑 expansion 而非 re-encode 整个 corpus**. 这是 cross-model embedding lifecycle 的另一种解：不存储 embedding 本身，存储**模型生成的 text augmentation**，模型换了重 augment，inverted index 用 lexical matching 仍 work. SPLADE 同理——sparse weight vocabulary-aligned 而非 model-specific dense vector，**模型升级 cost 更低**.

## Cited Pages

- [concepts/splade-family-baselines.md](../../concepts/splade-family-baselines.md)
