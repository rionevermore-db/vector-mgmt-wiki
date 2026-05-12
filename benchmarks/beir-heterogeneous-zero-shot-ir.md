---
title: BEIR（Heterogeneous Benchmark for Zero-shot IR Evaluation）
type: benchmark
sources: [thakur-2021-beir, formal-2021-splade-v2, santhanam-2022-colbertv2, muennighoff-2023-mteb]
related: [mteb-massive-text-embedding-benchmark.md, ../concepts/splade-sparse-retrieval.md, ../concepts/colbertv2.md, ../concepts/clip.md, ../topics/sparse-dense-hybrid-retrieval.md]
created: 2026-05-12
updated: 2026-05-12
---

# BEIR: A Heterogeneous Benchmark for Zero-shot IR Evaluation

**TL;DR**: Thakur + Reimers et al. NeurIPS 2021 [thakur-2021-beir] (UKP Lab + HuggingFace) 提出 **information retrieval (IR) zero-shot 评估标准 benchmark**——18 个公开数据集 + 10 个 retrieval system (lexical / sparse / dense / late-interaction / re-ranking architectures) 系统化对比 OOD generalization. **对 wiki 内 vector DBs 的核心价值**: (1) **wiki 内首个 retrieval-side standard benchmark**——MTEB (Muennighoff 2023) 是 embedding model benchmark, **BEIR 是 retrieval system benchmark** (评估完整 retrieval pipeline 而非仅 embedding); (2) **被 wiki 多 ingest cited but un-ingested**——SPLADE v2 §4 / ColBERTv2 §5 / MTEB §2.2 显式 cite BEIR 作 OOD evaluation, 之前 wiki 无 source 支撑; (3) **18 datasets across diverse domains**: 学术 (TREC-COVID, NFCorpus) / 财经 (FiQA) / 法律 / 客服 (CQADupStack) / Wikipedia (DBPedia, FEVER, NQ, HotpotQA) / 论坛 (Quora) / etc.; (4) **关键 finding**: **BM25 is a robust baseline**——OOD setting 下 dense retriever 通常 underperform BM25 (与 in-domain MS MARCO 相反); **re-ranking + late-interaction 平均最优 but high compute cost**. **Talk 实用价值**: BEIR 是 wiki 内 hybrid retrieval pipeline OOD evaluation 主标尺.

## 实验设置

[per thakur-2021-beir §2]

### 18 datasets（覆盖 diverse domains）

| Dataset | Domain | Task type |
|---|---|---|
| MS MARCO (in-domain only) | web search | passage retrieval |
| TREC-COVID | biomedical | retrieval |
| BioASQ | biomedical | retrieval + QA |
| NFCorpus | biomedical | retrieval |
| NQ (Natural Questions) | Wikipedia | QA |
| HotpotQA | Wikipedia | multi-hop QA |
| FiQA-2018 | finance | retrieval |
| Signal-1M (RT) | social media | retweet prediction |
| TREC-NEWS | news | retrieval |
| Robust04 | news | retrieval |
| ArguAna | argumentation | counter-argument retrieval |
| Touché-2020 | argumentation | controversial Q's |
| CQADupStack (12 sub-datasets) | StackExchange | duplicate question |
| Quora | online Q&A | duplicate question |
| DBPedia | Wikipedia entities | retrieval |
| SCIDOCS | scientific docs | citation prediction |
| FEVER | Wikipedia | fact verification |
| Climate-FEVER | climate | fact verification |
| SciFact | scientific | claim verification |

### 10 retrieval systems evaluated

1. **BM25** (lexical sparse baseline)
2. **TF-IDF** (lexical sparse baseline)
3. **SBERT** (Sentence-BERT, dense)
4. **DPR** (Dense Passage Retrieval, dense)
5. **ANCE** (dense + hard negative mining)
6. **TAS-B** (dense + balanced topic-aware sampling)
7. **GenQ** (synthetic query generation)
8. **ColBERT** (late-interaction)
9. **doc2query-T5** (sparse + document expansion)
10. **BM25 + CE** (BM25 + cross-encoder rerank, hybrid)

### Evaluation protocol

- **Zero-shot** evaluation: 仅 MS MARCO 训练, **不在 BEIR datasets fine-tune**
- nDCG@10 主要 metric
- 1k queries / dataset / model

## 结果（updated production reality）

[per thakur-2021-beir §4-5 + community 2022-2025 updates]

**Paper-time 关键 findings (2021)**:
- **BM25 is robust baseline**: 仅 nDCG@10 avg 0.412, 但 18/18 datasets robust
- DPR: 0.327 avg (lower than BM25)
- TAS-B: 0.413 avg (similar to BM25)
- ColBERT: 0.453 avg (best dense + late-interaction)
- **BM25 + CE rerank: 0.467 avg (best hybrid)** at high compute cost

**Post-paper updates (2022-2025 community evaluation)**:
- SPLADE v2 (Phase 3 ingest): 0.464 avg (sparse SOTA in BEIR)
- ColBERTv2 (Phase Batch #1 ingest): 0.500 avg + best on 22/28 (after extension)
- BGE-M3 (Phase Batch #3 ingest): 多 axis OSS SOTA

→ BEIR 是 wiki 内 sparse + dense + late-interaction 各 algorithm 评估 anchor.

## 可信度评估

**优势**:
- 18 datasets 覆盖 diverse domains (vs MS MARCO single-domain)
- Open leaderboard (持续更新 community-vetted)
- Zero-shot setting reflect production OOD reality
- 10 baseline systems (lexical / sparse / dense / late-interaction / hybrid)

**Limitations**:
- 部分 datasets 不公开 (CQADupStack / BioASQ / Signal-1M / TREC-NEWS / Robust04) — 影响 reproducibility
- 主要 English (multilingual coverage 缺)
- Long-document retrieval 不足 (vs MLDR / LoTTE 补充)
- Sparse + dense + hybrid fusion 不在 single benchmark scope

## 与 wiki 内 ingest 关系

### BEIR vs MTEB 互补

[per thakur-2021-beir + muennighoff-2023-mteb]

| | BEIR | MTEB |
|---|---|---|
| Focus | retrieval system | embedding model |
| Datasets | 18 IR-specific | 58 (含 retrieval 子集) |
| Eval | nDCG@10 (retrieval) | 8 task types |
| Architectures evaluated | sparse + dense + late-interaction + hybrid | embedding model output quality |
| MTEB retrieval task ⊃ BEIR? | partial (MTEB 包含部分 BEIR datasets in retrieval task) | n/a |

→ BEIR (retrieval system) + MTEB (embedding model) 互补——production 选择时 BEIR 评估 pipeline OOD quality, MTEB 评估 embedding model quality.

### BEIR cited in wiki ingested papers

[per wiki ingest history]

- **SPLADE v2** (Phase 3) §4 detailed BEIR results table——DistilSPLADE-max 11/14 best
- **ColBERTv2** (Batch #1) §5——22/28 best with 8% relative improvement
- **BGE-M3** (Batch #3) §4——18 languages BEIR-multilingual extension
- **MTEB** (Batch #2) §2.2——BEIR datasets as MTEB retrieval task subset

→ BEIR 是 wiki 内 sparse/dense/late-interaction retrieval **共通 evaluation framework**.

## Open Questions

- **BEIR extension to multimodal**: 当前 text-only, multimodal IR benchmark equivalent 不存在
- **BEIR + production reranking workload**: BM25 + CE 是 BEIR best but compute cost high, 实际 production trade-off case 不公开
- **BEIR + hybrid fusion**: BEIR 评估 individual system, hybrid fusion (RRF / α-blend) systematic comparison 不在 scope
- **BEIR + filter / scalar workload**: BEIR pure retrieval, attribute filter integration 不评估
- **Long-doc BEIR extension**: LoTTE / MLDR fill some gap but BEIR 自身未扩展

Cited by: 待 query 引用
