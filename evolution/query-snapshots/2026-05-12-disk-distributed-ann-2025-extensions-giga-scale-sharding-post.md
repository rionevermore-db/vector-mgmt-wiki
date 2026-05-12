---
query-key: giga-scale-sharding
date: 2026-05-12
phase: post
ingest-context: disk-distributed-ann-2025-extensions
wiki-pages-total: 90
cited-pages: [concepts/disk-distributed-ann-2025-extensions.md, concepts/frontier-2025-distributed-vector-search.md]
cited-count: 2
---

# Post-snapshot (disk-distributed-ann-2025-extensions): giga-scale-sharding

## TL;DR

**重大 NEW** (talk 主题强相关): BatANN 给出 wiki **第二个 OSS 学术 distributed disk-based 分布式 ANN benchmark 数据点**——100M-1B vectors / 10 servers / <6ms 均延迟 Recall@10=0.95 / 6.49× over scatter-gather. 与 SPIRE (Ingest #13) 形成 **对立 design choice**:
- **BatANN 路线**: 16-node + 1TB RAM 千亿配置 → 保 single global graph + baton-passing 跨 node query state, 高 locality 低 RPC
- **SPIRE 路线**: 16-node + 1TB RAM 千亿配置 → 弃 single graph + balanced hierarchical multi-level + end-to-end accuracy

production 选哪个取决于:
- query 模式: 异质 batch → SPIRE 优势; 同质 batch → BatANN 优势
- 网络: 高带宽低延迟 → BatANN 受益; 一般网络 → SPIRE 受益
- fidelity 容忍度: 极高 fidelity 要求 → BatANN; 可容忍 partition 路由小误差 → SPIRE

关键 NEW: **千亿规模主流方案现在有两条 OSS academic 路径** (vs 之前 wiki 仅 Pinecone/Meta 闭源大规模数据点).

## Cited Pages

- [concepts/disk-distributed-ann-2025-extensions.md](../../concepts/disk-distributed-ann-2025-extensions.md)
- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
