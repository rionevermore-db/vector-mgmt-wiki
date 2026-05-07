---
source-url: https://github.com/milvus-io/milvus-docs
upstream-site: https://milvus.io/docs
fetched-at: 2026-05-07
acquisition-method: git-clone
commit-hash: f7f7c21f775aa7ec8aba9c3dca78cc7644fae362
version: v2.6.x
---

# Milvus 文档快照

`milvus-io/milvus-docs` 仓库 default branch 在 2026-05-07 的快照。

## 获取流程

1. 尝试 `https://milvus.io/llms-full.txt` 与 `/llms.txt` —— 都返回 404
2. Fallback 到 GitHub 文档仓库 [`milvus-io/milvus-docs`](https://github.com/milvus-io/milvus-docs)
3. `git clone --depth 1` 后 strip `.git/`（102 MB → 减到 116 MB 纯文档）
4. Commit hash `f7f7c21f775aa7ec8aba9c3dca78cc7644fae362` 用作 citation 锚点（v2.6.x 默认分支当时 HEAD）

## 与 SIGMOD 2021 论文 (wang-2021-milvus) 的关系

[wang-2021-milvus](../../papers/wang-2021-milvus.pdf) 描述的是 Milvus **1.x** 架构（C++ + Knowhere + Faiss-based + 单 writer/多 reader shared-storage）。本快照是 **v2.6.x** —— Milvus 2.0+ 是 cloud-native 重写：

- **存算分离**进一步深化（DataNode / QueryNode / IndexNode 多角色）
- **Log broker** 替代 SIGMOD 论文里的 single-writer 模型
- **Streaming service**（Woodpecker）是 2.x 引入的新组件
- **DiskANN / GPU CAGRA** 等被纳入索引选项（论文时未集成）
- **新功能层**：multi-tenancy、JSON / array / VARCHAR 字段、function-based 检索（embedding 函数内嵌）等

引用此 source 时应记 source key `milvus-docs` 而不是 `wang-2021-milvus`，避免混淆 1.x 与 2.x。

## 目录概览

```
site/en/
├── about/              ← overview, comparison, limitations, roadmap, adopters
├── reference/
│   ├── architecture/   ← architecture, data_processing, four_layers,
│   │                     main_components, streaming_service, woodpecker_architecture
│   └── sys_config/     ← system parameters
├── userGuide/          ← collection / data-import / search 等使用指南
├── adminGuide/         ← deploy / monitor / backup / clouds
├── getstarted/         ← Docker / K8s / GPU 部署
├── faq/                ← 常见问题
├── tutorials/          ← end-to-end 教程
├── integrations/       ← LangChain 等整合
├── embeddings/         ← embedding 函数支持
├── rerankers/          ← reranker 支持
└── release_notes.md    ← 版本变更记录
```

## 引用约定

- Citation 形如：`[per sources/docs/milvus/site/en/reference/architecture/main_components.md]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `milvus-docs`
- 文档版本可能演进；当 Milvus 升大版本（如 v3.x）时建议新建 `sources/docs/milvus-<YYYY-MM>/` 快照而不是覆盖
