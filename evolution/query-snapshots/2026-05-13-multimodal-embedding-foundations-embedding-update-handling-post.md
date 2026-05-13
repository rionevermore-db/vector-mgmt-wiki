---
query-key: embedding-update-handling
date: 2026-05-13
phase: post
ingest-context: multimodal-embedding-foundations
wiki-pages-total: 98
cited-pages: [concepts/multimodal-embedding-foundations.md]
cited-count: 1
---

# Post-snapshot (multimodal-embedding-foundations): embedding-update-handling

## TL;DR

**重大 NEW** (talk SIGMOD 2026 cross-model migration 主题直接关键):

**BLIP-2 Q-Former 模式 = wiki 内 first architectural-level cross-model migration 解**:
- "**Frozen image encoder + frozen LLM + 188M trainable Q-Former**"
- 升级 LLM (OPT→FlanT5→Llama→Qwen) 时, **只需重训 Q-Former, image encoder + corpus embedding 不动**——cross-model migration cost 降到只是 188M bridge 重训
- 是 unified-stack RAG 升级时的真正 architectural 解 (vs 全量 re-encode corpus)

**ImageBind 6-modality unified embedding** = 多模态 cross-model migration 复杂度新维度:
- 6 modality 每个升级 model 时, **要保持 6-modality shared space 一致性**——任 1 modality 升级 → 重 train 其他 5 modality alignment
- Cross-modality consistency 是新 axis cost

**ALIGN 1.8B noisy web data** = production embedding scale-up 现实:
- 升级 model 时 data preparation cost 也 scale ~1B——cross-model migration 不仅是 inference re-encode cost, 还含 training data cost

**关键 NEW**: cross-model migration 在多模态场景**比单模态严重 6-10×**——modality 数 × per-modality re-encode + cross-modality 一致性 train. 是 talk SIGMOD 2026 必须明确区分 single-modality vs multi-modality 的 migration cost.

## Cited Pages

- [concepts/multimodal-embedding-foundations.md](../../concepts/multimodal-embedding-foundations.md)
