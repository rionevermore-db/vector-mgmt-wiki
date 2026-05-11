---
source-url: https://github.com/qdrant/landing_page
upstream-site: https://qdrant.tech/documentation
fetched-at: 2026-05-11
acquisition-method: git-clone (sparse-checkout)
commit-hash: fa07b1b8c62a8e2fcff0c655dd564576da17e797
upstream-default-branch: master
upstream-updated-at: 2026-05-07T20:00:30Z
subdir: qdrant-landing/content/documentation
size: 11 MB / 2490 markdown files
qdrant-version-coverage: v0.8.0 (distributed) through v1.17.0 (filter HNSW disable) — most features released v1.x
---

# Qdrant 文档快照

`https://github.com/qdrant/landing_page` 仓库 `qdrant-landing/content/documentation/` 子目录，sparse-checkout at commit `fa07b1b8`（2026-05-07 upstream tip）。Qdrant 是 Rust 实现的开源 vector DBMS（Apache-2.0），Qdrant Solutions GmbH 维护。文档涵盖 Qdrant Engine（OSS）+ Qdrant Cloud / Hybrid Cloud / Private Cloud / Enterprise / Edge 五个 SKU。

## 获取流程

| 步骤 | 状态 |
|---|---|
| 1. `https://qdrant.tech/llms-full.txt` | **✗ 404** |
| 2. `https://qdrant.tech/llms.txt` | **✗ 404** |
| 3. `https://docs.qdrant.tech/llms*.txt` | **✗ 404** |
| 4. GitHub clone `qdrant/landing_page` (CLAUDE.md mapping) | **✓** |

→ 回落 path C 优先级 2：GitHub sparse-checkout。

## Repo 结构（仅 documentation 子目录被 checkout）

```
sources/docs/qdrant/
├── _index.md
├── overview/                  ← what-is-qdrant, vector-search basics
├── manage-data/               ← **核心**
│   ├── collections.md         ← Collections, named vectors, aliases (model upgrade)
│   ├── indexing.md            ← HNSW + Payload + Filterable HNSW + Tenant/Principal index + ACORN
│   ├── multitenancy.md
│   ├── payload.md
│   ├── points.md
│   ├── quantization.md        ← Scalar / Binary / 1.5/2-bit / Asymmetric / Product
│   ├── storage.md             ← in-memory vs memmap; payload InMemory vs OnDisk (RocksDB)
│   └── vectors.md
├── search/                    ← filter / hybrid / low-latency / relevance / text
├── search-precision/          ← LLM-automated filtering + reranking
├── distributed_deployment.md  ← Raft + sharding (consistent hash + user-defined)
├── cloud/                     ← Qdrant Cloud / RBAC / billing / scaling
├── hybrid-cloud/
├── private-cloud/
├── edge/
├── ecosystem-tab.md           ← integrations
├── frameworks/
├── tutorials-*/
├── release-notes.md
└── ... (其他)
```

## 主要章节

```
# overview/what-is-qdrant.md             ← 介绍 + 核心概念
# overview/vector-search.md              ← vector search 基础
# manage-data/collections.md             ← Collections, aliases
# manage-data/indexing.md                ← **HNSW + Filterable HNSW + ACORN + Payload index**
# manage-data/quantization.md            ← Scalar/Binary/1.5/2-bit/Asymmetric/PQ
# manage-data/storage.md                 ← in-memory / memmap / on-disk payload
# manage-data/multitenancy.md            ← payload partitioning + tenant index
# search/filtering.md                    ← must/should/must_not + range/match/geo
# search/hybrid-queries.md               ← multi-vector hybrid (dense+sparse)
# search/search.md                       ← search API + planner
# distributed_deployment.md              ← Raft + sharding + replication
# ...
```

## 与 wiki 已 ingest source 的关系

- **wang-2021-milvus Table 1 列 Qdrant 为开源对手** → 本 source 是 Qdrant 自家完整描述
- **patel-2024-acorn**: Qdrant v1.16.0 **集成 ACORN 算法**作为 filterable HNSW 的 fallback——这是 wiki 内 ACORN 首次 production deployment 证据
- **gollapudi-2023-filtered-diskann (FilteredVamana)**: Qdrant 的 Filterable HNSW 与 FilteredVamana 是不同 base 的 filter-aware build：**HNSW base + extra edges** vs **Vamana base + label-aware RobustPrune**
- **gao-2024-rabitq**: Qdrant 没集成 RaBitQ；当前 Binary Quantization 是 wiki 已有 quantization landscape 的 Qdrant-specific 工程实现
- **patel-2024-acorn + qdrant-docs**: ACORN frontier 关闭——production deployment 实证

## 引用约定

- Citation 格式：`[per sources/docs/qdrant/<path>.md §<section>]`
- 例：`[per sources/docs/qdrant/manage-data/indexing.md "Filterable HNSW Index"]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `qdrant-docs`
- Qdrant 版本演进快（每年多个 feature release），重大版本变化时新建 `sources/docs/qdrant-<YYYY-MM>/` 快照

## 关键版本声明

- **v0.8.0** 分布式部署（Raft + sharding）
- **v1.0** Filterable HNSW（payload-aware extra edges）
- **v1.7.0** User-defined sharding + Sparse vectors
- **v1.11.0** Tenant Index + Principal Index + On-disk payload index
- **v1.13.0** Resharding (**Cloud only**)
- **v1.15.0** 1.5-bit / 2-bit / Asymmetric Quantization
- **v1.16.0** **ACORN Search Algorithm 集成**（filterable HNSW fallback）
- **v1.17.0** Disable extra edges for specific payload fields
