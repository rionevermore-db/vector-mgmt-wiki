---
title: WarpSelect（GPU 上的 k-selection）
type: concept
sources: [johnson-2017-faiss-gpu]
related: [product-quantization.md, ../topics/gpu-vs-cpu-ann.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md]
created: 2026-05-07
updated: 2026-05-07
---

# WarpSelect

**TL;DR**: GPU 上的 k-selection（top-k 距离选择）算法，状态完全保留在寄存器中、单次扫描输入、用 odd-size bitonic 网络做 warp 内排序合并。在 Titan X 上达 55% 峰值带宽（k=100），比前作 fgknn 快 1.62–2.01×，把 GPU ANN 的瓶颈从 k-selection 移开。[johnson-2017-faiss-gpu §4]

## 提出背景

Jeff Johnson, Matthijs Douze, Hervé Jégou（Facebook AI Research），Faiss-GPU 的核心算法之一。

历史问题：ANN 的最后一步是 top-k 选择，CPU 上常用 max-heap，但 GPU 上的 heap 实现退化严重：

- **Warp divergence**：插入路径数据相关，每 lane 走的分支不同；
- **Data-dependent memory movement**：heap 的 sift-up / sift-down 在不同位置触发不同次数 swap；
- **Shared memory pressure**：状态在 shared memory 而非 register；
- 结果：前作 fgknn / TBiS / GPU heap 都达不到内存带宽峰值的合理比例。

WarpSelect 重新设计 k-selection 以贴合 GPU 的 SIMT + register-rich 架构。[johnson-2017-faiss-gpu §3.3]

## 算法核心（Algorithm 3）

### 数据结构

每个 warp（32 lanes）负责**一个** query 的 k-selection：

- **Thread queue** T_j（lane-local，t 个元素，从大到小）：每 lane 自己的"粗过滤队列"
- **Warp queue** W（warp-shared lane-stride array，k 个元素，从小到大）：当前已知 k 个最小

两者都用 lane-stride register array（lane j 在 register r 存第 r·32+j 个元素），全寄存器，零 shared memory。[johnson-2017-faiss-gpu §3.2 + §4.2]

### 单次扫描更新

对每 32 个新元素 a_{32i+j}（j 是 lane id）：

1. 若 a > T₀ʲ（lane j 自己 thread queue 头部）→ 一定不在最小 k 内，丢弃
2. 否则 insertion sort 入 thread queue
3. 用 **warp ballot** 检测：是否有任何 lane 的 T₀ʲ < W_{k-1}（全局最大 W）？
4. 若有 → ODD-MERGE 合并 thread queues 与 warp queue → 重建 invariant

每 group 32 个 input 平均只 1–3 次常数时间操作，成本被距离计算 amortize。

### Bitonic 排序网络（适配非 2 次幂）

论文 Alg 1（MERGE-ODD）+ Alg 2（SORT-ODD）：

- 起源 Batcher 1968 bitonic sort，本论文改造为支持任意大小（不要求 2 的幂）；
- COMPARE-SWAP 用 warp shuffle（CUDA `__shfl_xor`）实现 lane 间数据交换；
- 跨 32 lanes 的 swap 通过 warp shuffle，lane 内的 swap 在 register array 内寻址。

## 复杂度（附录证明）

输入长度 ℓ，warp size w=32，thread queue 大小 t：

- N₁ = ℓ/32 次扫描
- N₂ = O(k·log(ℓ)/w) 次 lane 内 insertion sort 触发
- N₃ = O(k·log(ℓ)/t) 次 warp 全 ODD-MERGE 触发

总成本 N₁·C₁ + N₂·C₂ + N₃·C₃，C₂ 与 C₃ 都是 t 的函数；t 选择是 trade-off：

| k | t |
|---|---|
| ≤32 | 2 |
| ≤128 | 3 |
| ≤256 | 4 |
| ≤1024 | 8 |

[johnson-2017-faiss-gpu §4.3 + Appendix]

## 关键性质

| 维度 | WarpSelect | fgknn / TBiS / GPU heap |
|---|---|---|
| 状态位置 | **寄存器** | shared memory / global |
| 输入扫描次数 | 1 | 多 |
| Warp divergence | 极少（ballot 控制） | 高（heap path） |
| 与距离 kernel 融合 | **是** | 否 |
| k 上界 | 1024 | 通常更小 |
| 速度（k=100, Titan X） | **1.62× fgknn**, 55% 峰值带宽 | baseline |
| 速度（k=1000） | 2.01× fgknn, 16% 峰值 | baseline |

[johnson-2017-faiss-gpu Fig 3]

## 与距离 kernel 的融合（最大杠杆）

WarpSelect 的最大价值不在算法本身，而在**它能 fuse 到距离计算的 kernel 里**：

- **精确搜索**：cuBLAS GEMM 算 -2x·y 后，fused kernel 把 ||y||² 加到结果**直接送入 WarpSelect 寄存器**，无需写回中间矩阵 D'。论文实测节省 25% 时间。[johnson-2017-faiss-gpu §5.1]
- **IVFADC**：扫倒排表的 lookup-add kernel 里直接 fuse k-selection；写回带宽是真正瓶颈，所以 WarpSelect 的"不写回"特性极有价值。[johnson-2017-faiss-gpu §5.3]

## 典型实现

- Faiss `IndexFlatGpu`、`IndexIVFPQGpu` 内部 k-selection 即 WarpSelect；
- CUDA C++ template 实现，针对不同 k 范围编译特化版本。

## Open Questions

- **k > 1024 不支持** —— 当前设计的 register pressure 上限。大 k（千万级 retrieval 中常见）需要分级 selection 或不同算法。
- **k=1000 时只 16% 峰值带宽** —— 论文坦承大 k 性能掉，但根因（register pressure / warp queue 排序成本）未充分分析。
- **新 GPU 架构（Hopper、Blackwell）**：register file 与 warp shuffle 行为变化对 WarpSelect 的最优 t 配置影响如何？论文是 Pascal / Maxwell 时代设计。
- **跨 warp k-selection**：当前一 warp 处理一 query；有些场景一 query 太大需要多 warp 协作，WarpSelect 不直接支持。
