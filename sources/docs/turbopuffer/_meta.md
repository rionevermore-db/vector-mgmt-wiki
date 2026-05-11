---
source-key: turbopuffer-docs
title: Turbopuffer 官方文档（llms-full.txt + llms.txt）
vendor: Turbopuffer
source-url: https://turbopuffer.com/docs
acquisition-method: llms-full-txt + llms-txt
fetched-at: 2026-05-11
files:
  - llms-full.txt   # 15803 行 / 537 KB — 完整 doc prose (architecture / concepts / tradeoffs / vector / hybrid / fts / write / query / recall / pinning / byoc / security / cmek / regions / limits / pricing-log)
  - llms.txt        # 52 行 / 4 KB — overview prefix + URL catalog
version-coverage: Turbopuffer GA + April 2026 pricing/pinning era (closed-source commercial SaaS, no version tag公开)
notes: |
  Turbopuffer (turbopuffer.com) is closed-source commercial SaaS. No GitHub docs repo, no OSS.
  llms-full.txt is hybrid format: full prose for all `<section>...</section>` tags.
  Key novelty: SPFresh production deployment (centroid-based ANN, Microsoft Research SOSP 2023);
  object-storage-native architecture (S3 as primary durable layer);
  compute-compute separation (query nodes + indexing nodes both stateless, auto-scaled);
  LSM tree natively on object storage (unique vs all peer DBMS);
  3.5T+ docs / 13PB+ total / 100M+ namespaces production scale claim.
  Compared to other path C ingests:
    Pinecone (llms.txt + llms-full.txt) — closed SaaS peer, opaque slab vs Turbopuffer transparent design
    Milvus (GitHub docs) — OSS Go vs Turbopuffer closed-Rust
    Qdrant (GitHub sparse-checkout) — OSS Rust binary vs Turbopuffer closed-Rust SaaS
    Weaviate (llms.txt + 5 twin pages) — OSS Go vs Turbopuffer closed-Rust
    Vespa (llms-full.txt 92K + llms.txt 966) — OSS C++/Java vs Turbopuffer closed-Rust
---

# Turbopuffer Documentation Source

This directory holds the complete Turbopuffer documentation acquired via the `llms-full.txt` + `llms.txt` LLM-friendly endpoints on 2026-05-11.

## Acquisition

- Path C priority 1 (llms-full.txt / llms.txt) — both hit. No GitHub docs needed.
- llms.txt: short catalog prefix (52 lines)
- llms-full.txt: complete prose docs (15803 lines / 537 KB) covering all sections wrapped in `<section>...</section>` tags

## Key sections (anchor for citations)

Use line range or `<section>` tag for citations:

- `<architecture>` (lines 1-135) — system design, cache hierarchy, WAL, indexing nodes, cold/warm latency
- `<concepts>` (lines 2242-2501) — glossary including SPFresh, WAL, LSM, multi-tenancy
- `<guarantees>` (lines 4410-4462) — durable writes, consistent reads, atomic conditional writes, ACID
- `<hybrid>` (lines 4464-5497) — hybrid search architecture (vector + BM25)
- `<index>` (lines 5498-5539) — introduction / value prop
- `<limits>` (lines 5541-5605) — production scale numbers (3.5T+ docs, 100M+ namespaces, etc.)
- `<performance>` (lines 6083-6183) — optimization guide
- `<pinning>` (lines 6606-6956) — reserved compute per namespace (April 2026 feature)
- `<pricing-log>` (lines 6957-7003) — pricing changes history
- `<query>` (lines 7077-10383) — full query API including filtering, multi-query, ranking
- `<recall>` (lines 11140-11352) — recall measurement endpoint + continuous recall monitoring
- `<tradeoffs>` (lines 12250-12298) — design tradeoffs and the "excels at / not best fit" matrix
- `<vector>` (lines 12368-13108) — vector search guide
- `<write>` (lines 13234-end) — write API including conditional writes, atomic batches

## Notes on long-term stability

- Turbopuffer is closed-source and rapidly evolving. The April 2026 fetch represents snapshot state.
- Future re-fetches should go to `sources/docs/turbopuffer-<YYYY-MM>/` to preserve history (per CLAUDE.md path C versioning rule).
