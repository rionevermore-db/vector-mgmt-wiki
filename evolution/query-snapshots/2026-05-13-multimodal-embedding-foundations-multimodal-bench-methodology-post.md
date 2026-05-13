---
query-key: multimodal-bench-methodology
date: 2026-05-13
phase: post
ingest-context: multimodal-embedding-foundations
wiki-pages-total: 98
cited-pages: [concepts/multimodal-embedding-foundations.md, concepts/clip.md, concepts/siglip.md]
cited-count: 3
---

# Post-snapshot (multimodal-embedding-foundations): multimodal-bench-methodology

## TL;DR

**重大 NEW** (talk multimodal-bench-methodology query 多 snapshot 标记的持续 wiki gap 显著弥补):

**1. 多模态 embedding lineage 5-paper 完整** (wiki 现):
- ALIGN (2021, scale-first) + CLIP (2021, curated-scale) → SigLIP (2023, training efficiency) → BLIP-2 (2023, frozen + bridge) → ImageBind (2023, 6-modality)
- 这 5 model **正是 talk 多模态 benchmark methodology fair comparison baseline**

**2. ImageBind 6-modality 部分填补 multimodal benchmark gap**:
- audio + depth + thermal + IMU 3 个新 modality 现 source-backed (vs 之前 wiki 仅 image+text)
- spatial (lat/lon) 不在 ImageBind 6, 但 **modality binding pattern 可推广**——(image, geo-tile) pair-train + emergent (text, geo) zero-shot alignment
- 是 talk **三模 retrieval (vector + scalar + spatial)** algorithm-level 新解: 把 spatial 作为 ImageBind 7th modality

**3. 3 模检索 algorithm-level 路径明确**:
- (a) **多 system 拼接**: vector DB + spatial DB + scalar filter in app layer (production 默认)
- (b) **Unified algorithm**: ImageBind-style 多 modality embedding 通过 image-as-bridge + spatial-as-7th-modality 学到 unified embedding
- 路径 (b) 是 algorithm-level 新可能, **production vendor zero 实现**, academic 仍在 ImageBind 6-modality 验证 stage

**4. 多模态 benchmark methodology fair-comparison axis 完整**:
- Fix encoder: CLIP / SigLIP / ALIGN / ImageBind / BLIP-2 (5 选 1)
- Fix training data: noisy web (ALIGN/ImageBind) vs curated (CLIP) vs LLM-distill (BLIP-2)
- Fix modality count: 2 (image-text) vs 6 (ImageBind) — 大 modality 越多则 storage 越大
- Fix downstream: zero-shot classification / cross-modal retrieval / VQA (BLIP-2) / embedding arithmetic (ImageBind)

**关键 NEW**: 多模态 benchmark gap 不是 "no benchmark" 而是 "**no unified 5-paper × 4-axis fair comparison standard**"——wiki 现可指导 talk 中如何构造 fair multi-modal retrieval bench.

## Cited Pages

- [concepts/multimodal-embedding-foundations.md](../../concepts/multimodal-embedding-foundations.md)
- [concepts/clip.md](../../concepts/clip.md)
- [concepts/siglip.md](../../concepts/siglip.md)
