---
query-key: multimodal-bench-methodology
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md]
cited-count: 1
---

# Post-snapshot (modern-embedding-paradigms): multimodal-bench-methodology

## TL;DR

无直接影响——3 paper 都是 text-only embedding. 间接 NEW: NV-Embed paradigm (decoder-LLM-as-embedder) 自然扩展到 multimodal——VLM (LLaVA / Qwen-VL) 可同方式作 multimodal embedder, NV-Embed 的"latent attention pooling + 移除 causal mask + instruction tuning"配方在 multimodal 适用. 但 wiki 目前**仍无 multimodal embedder concept** (vs text-only 已 5+ paper). Multimodal benchmark gap 持续——MTEB equivalent for image/audio retrieval 不存在.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
