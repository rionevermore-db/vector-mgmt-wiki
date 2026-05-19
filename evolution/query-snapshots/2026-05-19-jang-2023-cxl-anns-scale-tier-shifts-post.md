---
query-key: scale-tier-shifts
date: 2026-05-19
phase: post
ingest-context: jang-2023-cxl-anns
wiki-pages-total: 99
cited-pages: [systems/cxl-anns.md, topics/disk-vs-memory-ann.md, topics/gpu-vs-cpu-ann.md, systems/distributedann.md]
cited-count: 4
---

# Post-snapshot (jang-2023-cxl-anns): scale-tier-shifts

## TL;DR

**直接影响"存储介质"维度的质变图**。CXL-ANNS 在 billion-scale 一档新增一个质变选项：**单机 DRAM → CXL 解耦内存池**——不是同方案参数微调，而是换硬件 substrate（Type-3 EP，≤4095 EP/≤4 PB）。质变三轴现在：
- **索引选择**：CXL-ANNS 不改算法（仍 NSG graph），改的是数据驻留层
- **存储介质**：DRAM → CXL.mem（DRAM-SSD 之间新层，但 CXL-ANNS 把全精度全放内存池而非下放慢介质）→ 与 DiskANN 的 DRAM→SSD 质变并列的**另一种 billion-scale 质变方向**
- **并行能力**：质变在 EP-side——near-data DSA 把距离计算从 CPU 卸到内存设备（distance calc 降 119.4×），瓶颈从内存容量转移到 EP PE 算力（multi-host 6 host 时 PE 成瓶颈）

迁移点：billion-scale 全精度无损需求 + 有 CXL 硬件 → 切 CXL-ANNS；否则 DiskANN/SPANN（牺牲 latency）或 PQ（牺牲 recall）。gpu-vs-cpu-ann.md 新增"近数据计算 vs 加速器卸载"也是并行能力维度的质变对撞。

## Cited Pages

- [systems/cxl-anns.md](../../systems/cxl-anns.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)（三条 scale 轴对比表）
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)（近数据计算 vs 加速器卸载）
- [systems/distributedann.md](../../systems/distributedann.md)
