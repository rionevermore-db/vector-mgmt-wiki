---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: modern-embedding-paradigms
wiki-pages-total: 87
cited-pages: [concepts/modern-embedding-paradigms.md, concepts/bge-m3.md]
cited-count: 2
---

# Post-snapshot (modern-embedding-paradigms): embedding-update-handling

## TL;DR

**重大 NEW** (talk live demo 直接相关): 三 paradigm 给出 cross-model migration 不同 cost profile.
1. **GTE → NV-Embed-v2 升级**: 768d → 4096d, 维度不兼容 → 全量 re-encode 必需, cost = 1.5T × 50ms (NV-Embed) ≈ 25,000 GPU-hour
2. **Gecko-768 → NV-Embed-v2 升级**: 768d → 4096d, dim mismatch + 不同模型 backbone, 仍需 full re-encode
3. **同 paradigm 内升级** (NV-Embed-v1 → v2): 4096d unchanged, **可能用 LLM distillation 让 v2 "翻译" v1 embedding** (FRet-style) — 但论文无实验证据
4. **synthetic data 改写**: Gecko FRet 范式提示 "doc 端" embedding 升级时, 旧 corpus 不需重 encode 而是用 LLM 生成新 training pairs，**training-side migration ≠ encoding-side migration**, talk SIGMOD 2026 live demo 主题真正定位.

关键 NEW: cross-model migration 成本 = (dim mismatch?) + (训练数据相似度) + (model capacity 跨度) 三 axis 共同决定; 这三 paradigm 一年内频繁迭代 → 每 6-12 月就需 re-encode, **MTEB SOTA 进步速度 ~3-6 点/年 = 业界平均 corpus 寿命 < 1 年**.

## Cited Pages

- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
- [concepts/bge-m3.md](../../concepts/bge-m3.md)
