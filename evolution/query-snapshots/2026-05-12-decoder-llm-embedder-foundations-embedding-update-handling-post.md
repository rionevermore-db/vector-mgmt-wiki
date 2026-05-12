---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: decoder-llm-embedder-foundations
wiki-pages-total: 89
cited-pages: [concepts/decoder-llm-embedder-foundations.md, concepts/modern-embedding-paradigms.md]
cited-count: 2
---

# Post-snapshot (decoder-llm-embedder-foundations): embedding-update-handling

## TL;DR

**重大 NEW** (talk live demo 直接相关): 3 paper 给出 cross-model migration 三种**架构性应对**:
1. **E5-Mistral 路线** (合成数据): embedding model 升级时 GPT-4 重生成训练数据 + 重训 < 1k 步, **训练 cost 极低**, 但仍需 corpus 全量 re-encode
2. **GritLM 路线** (统一 model): "embedding model" 概念**消失**——如果 RAG stack 用同 LLM 做 retrieval + generation, 升级 LLM 时 retrieval 自动同步, **不存在 cross-model migration 问题**; talk 论文主题在此架构下不适用
3. **LLM2Vec 路线** (decoder→encoder 转换 recipe): 升级 LLM (e.g. Llama2 → Llama3 → Llama4) 时, 直接套用 3 步 recipe 改造, **不依赖 OpenAI / Microsoft 等私有 LLM**, public-only 路径更稳

关键 NEW: **GritLM 让 cross-model migration problem 在该架构下消失** = talk 论文核心 contribution gap 的对立设计哲学, 需在 talk 中明确区分两类架构 (separated vs unified).

## Cited Pages

- [concepts/decoder-llm-embedder-foundations.md](../../concepts/decoder-llm-embedder-foundations.md)
- [concepts/modern-embedding-paradigms.md](../../concepts/modern-embedding-paradigms.md)
