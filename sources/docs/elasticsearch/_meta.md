---
vendor: Elasticsearch (Elastic)
source-url: https://www.elastic.co/guide/en/elasticsearch/reference/current/
fetched-at: 2026-05-12
acquisition-method: WebFetch AI summary (knn-search 主 page; 完整 GitHub elastic/elasticsearch-docs 未 git-clone)
license: 文档公开可读 (Elastic License 不影响 docs)
notes: |
  Elasticsearch docs 量级巨大 (含 ES + Kibana + Beats + Logstash). Path C 退化为 WebFetch single-page summary——仅抓 kNN search 主 page 作 citation anchor.
  ELSER (Elastic Sparse Encoder) 是 ES 自有 neural sparse model, 未在本快照覆盖. 未来 ingest 可扩展.
---

# Elasticsearch docs snapshot

- `knn-search.md` — kNN search 主 page, HNSW + dense_vector + 量化 + hybrid + Lucene filter
