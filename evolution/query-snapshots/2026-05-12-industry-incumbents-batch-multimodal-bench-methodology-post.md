---
query-key: multimodal-bench-methodology
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: []
cited-count: 0
---

# Post-snapshot (industry-incumbents-batch): multimodal-bench-methodology

## TL;DR

**重大 NEW (反向)**——industry incumbent vendor 中 **0 个 multimodal-native**——Snowflake / Databricks / Elasticsearch / OpenSearch / Redis / MongoDB Atlas 都仅讨论 text + scalar filter, **未提及空间 / 图像 / 视频 retrieval first-class 支持**.

这强化 wiki "multimodal benchmark gap" 持续: 即使 6 大主流 production vendor 都不支持空间能力, 业界 multimodal retrieval **基础设施级别空白**仍未填.

production multimodal retrieval 必须自建 (e.g. 用 vector DB 存 CLIP image embedding + 在 application 层 join geospatial DB), 是 **跨 system 工程问题** 不是单 vendor 能解决.

## Cited Pages

(无)
