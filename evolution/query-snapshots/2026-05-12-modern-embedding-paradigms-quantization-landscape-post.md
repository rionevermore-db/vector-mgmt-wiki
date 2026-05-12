---
query-key: quantization-landscape
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md, concepts/matryoshka-embedding.md]
cited-count: 2
---

# Post-snapshot (modern-embedding-paradigms): quantization-landscape

## TL;DR

NEW indirect: NV-Embed paper §A 含 LLM-based embedding **pruning + quantization + knowledge distillation** 应用——基于 Llama3.2-3B / Qwen2.5-3B / Minitron-4B 压缩 SOTA 7B embedding model. 关键 NEW: 之前 wiki quantization 主要谈 PQ/OPQ/RaBitQ (vector-level), NV-Embed 引入**model-level compression for embedding generation** 新 axis——与 vector quantize 是不同 layer 的压缩 (model → embedding-generation 一步, vector quantize → embedding-stored 一步, 两 axis 可叠加). Gecko 256-dim 直接训出 compact embedding 是 **train-time dim quantization** (与 MRL 相通); 总体三阶段独立 compression:
1. Model compression (pruning + KD, train-time)
2. Output dim compression (Gecko 256d, MRL prefix, train-time)
3. Vector-level quantization (PQ/OPQ/RaBitQ, post-hoc)

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
