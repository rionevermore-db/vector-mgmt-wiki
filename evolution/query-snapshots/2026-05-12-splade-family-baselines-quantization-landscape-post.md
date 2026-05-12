---
query-key: quantization-landscape
date: 2026-05-12
phase: post
ingest-context: splade-family-baselines
wiki-pages-total: 86
cited-pages: [concepts/splade-family-baselines.md]
cited-count: 1
---

# Post-snapshot (splade-family-baselines): quantization-landscape

## TL;DR

间接影响——sparse retrieval 没有 PQ/OPQ/RaBitQ 类 vector quantization，但 DeepImpact paper 介绍了**学习 BM25-equivalent 整数权重**作为 sparse 压缩 (impact score quantized to int8 → bytes per posting)，这是 sparse 维度的"quantization 类比". 关键 NEW: dense 维 quantize float vector，sparse 维 quantize impact score——**两套 axis 独立**，hybrid 系统总压缩率 = sparse_compress × dense_compress.

## Cited Pages

- [concepts/splade-family-baselines.md](../../concepts/splade-family-baselines.md)
