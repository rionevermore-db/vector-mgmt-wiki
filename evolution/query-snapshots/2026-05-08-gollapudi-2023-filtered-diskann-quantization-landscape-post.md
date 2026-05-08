---
query-key: quantization-landscape
query: "PQ / OPQ / SQ / RaBitQ 等向量压缩方法的精度-速度-内存权衡？工业系统通常怎么和 IVF / HNSW 组合？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [concepts/product-quantization.md, concepts/filtered-vamana.md, systems/diskann.md]
cited-count: 3
---

# Post-snapshot (gollapudi-2023-filtered-diskann): quantization-landscape

## TL;DR (delta from yang-2020-pase post)

**Filtered-DiskANN 复用 DiskANN 的 PQ + 全精度 SSD re-rank 方案**——不引入新 quantizer，但**在 filter-aware 场景中 PQ 表现比 IVF-PQ 更好**。原因：filter 过滤大量候选后，剩余候选需要精确 ranking——DiskANN 的"PQ DRAM 导航 + SSD 全精度 re-rank"两阶段比"IVF-PQ 全 PQ"两阶段在 selective filter 下天然优——后者 ADC 失真在小候选集上累积，前者全精度 re-rank 不受 filter selectivity 影响。

## Answer

### Filtered-DiskANN 的 quantization 复用策略（NEW）

[per concepts/filtered-vamana.md "在 [DiskANN] 框架下：Filtered-DiskANN" + benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]

Filtered-DiskANN 没引入新 quantizer——直接复用 [DiskANN 论文](../../systems/diskann.md) §3 的双阶段：
- **PQ codes in DRAM**：用 PQ 估算距离作为 graph 遍历导航
- **SSD 全精度 re-rank**：每跳读 SSD 4KB block 获得全精度 vector，最终 top-k 用全精度精化

但**应用到 filter-aware** 场景时优势更大：
- IVF-PQ inline-processing：filter 过滤后剩 N' 个候选，全部用 PQ ADC ranking → 累积失真
- Filtered-DiskANN：filter 过滤后剩 N' 个候选用全精度 re-rank → recall 不受 filter selectivity 影响

### 实证数据（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "主结果 1-3"]

| Filter specificity | IVF inline-processing recall@10 | FilteredVamana recall@10 |
|---|---|---|
| 100pc | 90%+ | 95%+ |
| 50pc | 80%+ | 95%+ |
| 25pc | 60% | 95%+ |
| **1pc** | **<40%** | **>90%** |

→ **selectivity 越低（filter 越严），quantization 路径越退化**——IVF-PQ ADC 失真在小候选集上 dominate；DiskANN 全精度 re-rank 路径稳定。

### Quantization landscape 中"何时用 PQ" 的更新

之前 wiki 总结："PQ 节省内存换部分精度"。Filtered-DiskANN 加新维度：

| 场景 | 推荐 quantization |
|---|---|
| Static 大数据 + 内存敏感 | PQ in IVFADC（Faiss 经典） |
| MIPS + 高 recall | ScaNN anisotropic PQ |
| OLAP 大数据 + 几何剪枝 | VGPQ |
| **Filter-heavy + selective filter** | **DiskANN 双阶段（PQ DRAM 导航 + SSD 全精度 re-rank）** |
| Production high-recall | 全精度（SPANN / SPFresh / FilteredVamana） |
| Per segment 选 | Milvus segment-level index_type |

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| Quantization 维度 | PASE 8KB page-aligned 全精度 | **+ 双阶段 PQ + 全精度 re-rank 在 filter-heavy 优势** |
| IVF-PQ vs DiskANN PQ 对比 | 各场景独立 | **filter-heavy 场景 DiskANN 路径全程优于 IVF-PQ** |
| Filter selectivity 与 quantization 的耦合 | 未涉及 | **明确：低 selectivity → quantization 失真累积** |

### Open / 未覆盖

- **VGPQ + filter-aware**：理论可行未实测；ADBV 4-plan + filter-aware partition 可能匹配但 wiki 未实证
- **ScaNN anisotropic + filter-aware**：MIPS + filter 组合 zero coverage
- **RaBitQ**：仍未覆盖

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [systems/diskann.md](../../systems/diskann.md)
