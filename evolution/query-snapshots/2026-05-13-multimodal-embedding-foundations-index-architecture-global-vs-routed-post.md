---
query-key: index-architecture-global-vs-routed
date: 2026-05-13
phase: post
ingest-context: multimodal-embedding-foundations
wiki-pages-total: 98
cited-pages: [concepts/multimodal-embedding-foundations.md]
cited-count: 1
---

# Post-snapshot (multimodal-embedding-foundations): index-architecture-global-vs-routed

## TL;DR

NEW: ImageBind unified embedding space 暗示 **可全 6-modality 同 single index** (因 shared space)——production 选项:
- **(α) Single global multi-modal index**: ImageBind 6-modality 共享 embedding, 同一 HNSW/IVF 索引可容所有 modality, 拓扑选 (a)/(a')/(b)/(c)/(c') 之一
- **(β) Per-modality routed**: 每 modality 独立索引 + query 时 router 选目标 modality(ies)——传统 multi-modal RAG 默认

ImageBind 之前是 (β) 必然 (no unified space), ImageBind 后 (α) 成为 viable. **新 design space 维度** in wiki ANN 拓扑分类.

## Cited Pages

- [concepts/multimodal-embedding-foundations.md](../../concepts/multimodal-embedding-foundations.md)
