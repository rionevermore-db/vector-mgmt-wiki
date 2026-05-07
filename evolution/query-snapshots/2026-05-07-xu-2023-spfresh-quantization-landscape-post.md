---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [concepts/product-quantization.md, systems/spfresh.md, systems/spann.md, concepts/lire.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 5
---

# Post-snapshot (xu-2023-spfresh): quantization-landscape

## TL;DR (delta from milvus-docs post)

**SPFresh 强化"IVF 不必绑 PQ"的反例链条**：[SPFresh](../../systems/spfresh.md) = SPANN + LIRE，**全程全精度 + in-place update**。延续 [SPANN](../../systems/spann.md) 反例（IVF + 全精度 SSD posting），还多一个 update 维度。**Update 与 quantization 的耦合**也是开放问题：LIRE 假设 posting 全精度（继承 SPANN），融合 PQ 是否破坏 NPA？SPFresh 没解。

## Answer

### Quantization 与 update 路线的耦合（NEW，本次 ingest 暴露）

之前 wiki 的 quantizer 选择维度仅考虑 静态索引 的精度-内存-速度。**SPFresh ingest 后新增维度：与 update 路线的兼容性**。

| Quantizer | 与 in-place update 兼容 | 备注 |
|---|---|---|
| 全精度（SPANN-style） | **✓**（SPFresh 实证） | LIRE 假设全精度 + NPA |
| Scalar Quantizer (SQ8) | 待验证 | 单点量化，不影响 NPA 距离单调性 |
| Product Quantizer (PQ) | **未实证**——LIRE 假设全精度，融合 PQ 是否破坏 NPA？SPFresh 没解 | PQ 失真可能让"附近 posting NPA 检查"漏掉 violation |
| OPQ | 同 PQ | |
| ScaNN anisotropic | 同 PQ | |

→ **若需要 in-place update，全精度 SPANN-style 是已验证唯一路线**；PQ-based 系统（Faiss IVFPQ / DiskANN PQ）的 in-place 更新仍是开放问题。

### SPFresh 与 wiki 现有 quantizer 路线的对比

[per concepts/product-quantization.md "反例：SPANN 证明 IVF 不必绑 PQ"]：

之前 SPANN 反例：IVF + 全精度 SSD = 1B 单机 + 95% recall。
**SPFresh 升级反例**：IVF + 全精度 SSD + **in-place update** = 1B 持续 update + 资源 1% of DiskANN。

PQ 仍是值得用的（节省内存），但**当 update 是首要约束时，全精度路径更优**。这与 [DiskANN](../../systems/diskann.md) 路径形成另一种 trade-off：

| | DiskANN | SPANN | SPFresh |
|---|---|---|---|
| Quantizer | PQ DRAM + SSD 全精度 re-rank | 无（全精度 SSD） | 无（全精度 SSD） |
| Update | 周期 rebuild | 周期 rebuild | **in-place** |
| 内存预算 | 64 GB | ~32 GB | **~10 GB** |

### Open Questions（NEW）

- **LIRE + PQ 的兼容性**：PQ 失真是否破坏 NPA 必要条件？SPFresh 论文未验证；融合 PQ 后 LIRE 是否仍收敛？
- **OPQ / [ScaNN](../../concepts/scann.md) anisotropic + 增量更新**：所有量化文献假设静态 codebook；动态 update 下 codebook 是否需要 incremental retrain？
- **RaBitQ + in-place**：仍未覆盖

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/lire.md](../../concepts/lire.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
