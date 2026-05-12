---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: chen-2024-bge-m3
wiki-pages-total: 81
cited-pages: [concepts/bge-m3.md, concepts/clip.md]
cited-count: 2
---

# Post-snapshot (chen-2024-bge-m3): embedding-update-handling

## TL;DR (delta from mteb post)

**BGE-M3 simplify embedding upgrade 路径**——之前 production hybrid 需 3 个独立 model (SPLADE + CLIP + ColBERTv2), 各自独立 upgrade timeline. **BGE-M3 把 3 在 single model, upgrade frequency 降到 1×**. 关键 NEW: 当 BGE-M3 升级到 BGE-M4 (假设 future), 一次升级 cover 3 representation——比之前 3 独立 model 升级**减少 3× migration overhead**.

但 cross-model semantic preserve frontier (BGE-M3 → BGE-M4 / 或其他 model) 仍未关闭. Talk SIGMOD 2026 仍唯一 algorithm candidate.

## Cited Pages

- [concepts/bge-m3.md](../../concepts/bge-m3.md)
- [concepts/clip.md](../../concepts/clip.md)
