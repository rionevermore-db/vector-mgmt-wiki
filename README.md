# Vector Management Wiki

> 一个由 LLM 维护、围绕 **vector management for LLM inference** 的研究知识库。

## 这是什么

参考 [Andrej Karpathy 的 LLM-Wiki 模式](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 改造而成——把读论文 / 产品文档的认知劳动沉淀为**可互链、可检索、可演化的 markdown 知识库**。

三层架构：
- **Sources**：原始论文 / 产品文档，不可变
- **Wiki**：LLM 按 schema 生成的派生页面，可重建
- **Schema**（[`CLAUDE.md`](./CLAUDE.md)）：规则与契约，定义 Layer 2 长什么样

> "LLM 不是问答机器，而是知识簿记员。" 人负责策展、判断、提问；LLM 负责派生、互链、保鲜。

## 当前覆盖

- **算法**：HNSW / NSG / Vamana / IVF / PQ / OPQ / ScaNN / RaBitQ / WarpSelect / CAGRA ...
- **系统**：Faiss / Milvus / Pinecone / Qdrant / Weaviate / Vespa / Turbopuffer / DiskANN / SPANN / SPFresh / FreshDiskANN / AnalyticDB-V / PASE / VBASE / Starling ...
- **主题**：Disk vs Memory ANN · Attribute Filtering · Multi-Vector Queries · TopK vs Iterator · Index Selection ...
- **测评**：覆盖 Million / Billion / Trillion 三档规模的 benchmark
- **Queries**：已归档的高价值问题答案

全目录见 [`index.md`](./index.md)。

## 怎么用

### 1. 直接浏览（无需任何工具）
打开 [`index.md`](./index.md) 看全部 page 摘要 → 点进感兴趣的具体页面读细节。

### 2. 在本仓库提问（推荐 — 这是 wiki 真正的样子）
在仓库根目录开 Claude Code / Cursor / Aider 等，直接打技术问题：

- *"HNSW 和 NSG 都是 proximity graph，工程上怎么选？"*
- *"千亿/万亿向量在 16 节点私有云下如何分片？"*
- *"对比多个 vector DB 在向量+标量过滤下的性能时，如何公平 benchmark？"*

LLM 会从 wiki 检索 → 综合答案 → 标注 cited pages。**不是 hallucination 答案，是有 source 锚点的答案**。

### 3. 自己 fork 一份建你的领域

5 分钟 quick start：

```bash
git clone https://github.com/<user>/vector-mgmt-wiki my-wiki
cd my-wiki

# 1) 清空已有 wiki 内容（保留骨架）
rm -rf concepts/*.md systems/*.md topics/*.md benchmarks/*.md queries/*.md sources/papers/*
echo "_暂无。_" > index.md   # 或保留模板

# 2) 改 CLAUDE.md，把"vector management"替换为你的领域

# 3) 把第一篇 source 放进 sources/papers/，开 Claude Code：
#    > ingest <论文标题>
#    LLM 跑路径 A 自动找 URL + 下载 + ingest

# 4) 每天 1-2 小时持续 ingest，3 周后回头看你的 wiki
```

详细约束 + 5 种 page type 模板见 [`CLAUDE.md`](./CLAUDE.md)。

## 三种工作流

LLM 通过 [`CLAUDE.md`](./CLAUDE.md) 中定义的 schema 操作 wiki，三种典型动作：

| Workflow | 触发 | 作用 |
|---|---|---|
| **Ingest** | "ingest \<标题\>" / "ingest \<路径\>" / "ingest \<产品\> 文档" | 消化新 source，派生 5-15 个 page 改动 + 自动 commit |
| **Query** | 任何技术提问 | 从 wiki 综合答案，标注 cited pages |
| **Lint** | "lint" | 找 orphan / stale claim / contradiction / cross-link gap，定期保鲜 |

> 速成靠 ingest，保持专家靠 lint。

## 仓库结构

```
.
├── CLAUDE.md          ← Schema（最重要的单文件，是 wiki 的灵魂）
├── README.md          ← 本文件
├── index.md           ← 全 wiki 目录
├── log.md             ← append-only 操作日志
├── sources/           ← 原始论文 / 产品文档（不可变）
│   ├── papers/
│   └── docs/
├── concepts/          ← 算法 / 数据结构
├── systems/           ← 产品 / 工程系统
├── topics/            ← 跨概念主题
├── benchmarks/        ← 测评结果 + 方法论
└── queries/           ← 高价值 query 答案归档
```

## 关于本仓库的来源

由一次 "AI 知识管理" 主题 talk 衍生而来——演讲主线是 **"如何用 LLM 一日速成一个领域的专家"**。本仓库即演示用例：从零开始 3 周搭出一个 vector management 领域的研究 wiki。

slides + 详细 plan 见配套 talk material（联系仓库主可获取）。

## 给同事 / 访客的建议

如果你想自己搭一个 LLM-Wiki：

1. 先读完 [`CLAUDE.md`](./CLAUDE.md)（约 10 分钟）—— 这是整个机制的契约
2. 自己提一个**你领域里"专家级"的硬问题**作为 demo query
3. 列 10 篇必读 source，按优先级一篇篇 ingest
4. 每周 lint 一次，3 周后再问那个 demo query，看答案厚度
5. 让 LLM 主动提问（schema 第 5 条）——遇到归类不确定，**它会问你而不是自己拍板**

## License

- 内容（markdown / 图）：CC BY 4.0
- 脚本：MIT
- `sources/` 下文件版权归原作者，本仓库仅作 fair-use 学习引用
