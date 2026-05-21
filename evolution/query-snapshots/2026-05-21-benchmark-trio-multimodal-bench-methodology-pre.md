---
query-key: multimodal-bench-methodology
date: 2026-05-21
phase: pre
ingest-context: benchmark-trio
wiki-pages-total: 101
cited-pages: [topics/multimodal-embedding-retrieval.md, systems/vespa.md]
cited-count: 2
---

# Pre-snapshot (benchmark-trio): multimodal-bench-methodology

## TL;DR

当前 wiki 能指出**三模检索的能力不对称问题**,但**没有任何 benchmark 方法论 / harness 覆盖如何公平横测**。

能答的:
- **空间能力严重不对称** [per topics/multimodal-embedding-retrieval.md]:vector + scalar 成熟(5 策略),但空间维度几乎空白——只有 [systems/vespa.md] native 空间索引,多数 vendor 无 / 仅 bbox 近似。
- **multimodal embedding 端统一**:CLIP/SigLIP 把 image/text 映射到同一 cosine ANN 空间,所以"多模态"在 vector 层不需要特殊索引——但这≠"vector+scalar+空间"三模混合查询的 benchmark。
- wiki 已明确记 **head-to-head multimodal benchmark 不存在** 是行业 gap。

答不出的(当前盲点):
- **怎么公平测 cross-modal / OOD(query 与 base 分布不同)检索**——无 source、无标准数据集、无 recall 口径。
- **三模(vector+scalar+spatial)统一 benchmark harness** 完全无覆盖。
- 缺一个权威 benchmark(含 filter / sparse / OOD / streaming track)作锚点。

## Cited Pages

- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
