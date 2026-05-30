# Vector Management Wiki

> 一个由 LLM 维护、围绕 **vector management for LLM inference** 的研究知识库。

## 这是什么

参考 [Andrej Karpathy 的 LLM-Wiki 模式](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)：把读论文/产品文档的认知劳动，沉淀为可互链、可检索、可演化的 markdown 知识库；LLM 负责簿记，人负责策展。

## 怎么用（5 分钟 quick start）

### 浏览
- 直接打开 [`index.md`](./index.md) 看全 wiki 目录
- 想看具体主题：`concepts/`（算法）、`systems/`（产品）、`topics/`（跨概念主题）

### 自己 fork 一份建你的
1. Fork 本 repo，清空 `concepts/` `systems/` `topics/` `benchmarks/` `queries/` 下所有内容
2. 改写 [`CLAUDE.md`](./CLAUDE.md)：把"vector management"换成你的领域，调整 page type
3. 在 repo 根目录开 Claude Code 或 Cursor，把第一篇 source 丢进 `sources/papers/`，对 LLM 说 "ingest"
4. 每天 1-2 小时，3 周后回头看你的 wiki 长什么样

### 在本 repo 提问
在 repo 根目录开 Claude Code，直接提问。LLM 会优先从 wiki 里找答案并标注 citation。

## 仓库结构

详见 [`CLAUDE.md`](./CLAUDE.md) 的"目录结构"章节。


