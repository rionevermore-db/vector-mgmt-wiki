---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md, concepts/matryoshka-embedding.md]
cited-count: 2
---

# Post-snapshot (muennighoff-2023-mteb): quantization-landscape

## TL;DR

MTEB 评估 embedding model output quality, **不直接评估 quantization impact**. 关键 open: MTEB-evaluated embeddings (e.g., OpenAI text-emb-3-large 3072-d) 经 quantization (RaBitQ binary / MRL prefix truncation / PQ) 后, MTEB score 退化曲线 **wiki + 论文都 zero coverage**. Production 实际使用 quantized embedding 但 MTEB ranking 假设 full-precision——这是 wiki 内**"benchmark vs production deployment gap" 重要 frontier**.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
