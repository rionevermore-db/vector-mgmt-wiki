---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md, topics/multimodal-embedding-retrieval.md]
cited-count: 2
---

# Post-snapshot (muennighoff-2023-mteb): multimodal-bench-methodology

## TL;DR

MTEB **text-only**, 不评估 multimodal embedding (CLIP / SigLIP / ImageBind). 关键 wiki 内 multimodal benchmark frontier gap: text 有 MTEB / BEIR, multimodal 无 equivalent standard. MMTEB (multilingual MTEB) 扩展到 112+ languages 但仍 text-only. 关键 NEW: production multimodal embedding 选择缺乏 MTEB-style 客观标准 — 通常用 paper-specific benchmark (Flickr30K / MS-COCO image-text retrieval) 而非 universal.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
