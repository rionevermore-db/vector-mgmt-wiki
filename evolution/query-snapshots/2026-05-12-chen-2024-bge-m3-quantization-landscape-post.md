---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md]
cited-count: 1
---

# Post-snapshot (chen-2024-bge-m3): quantization-landscape

## TL;DR

BGE-M3 三 output 各自 quantization 哲学不同: dense 1024-d float 可 quantize (RaBitQ / PQ / int8); sparse 自然 sparse; colbert 走 ColBERTv2 residual compression. 关键 NEW: production 用 BGE-M3 需要 vendor 支持**三 quantization scheme 同时**——目前 wiki 仅 Vespa + Milvus 完整支持. BGE-M3 非 MRL-trained, 即 dense output 不支持 0-cost prefix truncation (vs OpenAI text-emb-3 / Voyage).

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
