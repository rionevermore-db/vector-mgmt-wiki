---
vendor: Redis Stack (Redis Inc.)
source-url: https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/
fetched-at: 2026-05-12
acquisition-method: WebFetch partial + 业界公知 (RediSearch 公开 design)
license: Redis Stack 当前 RSALv2 + SSPL (2024+); RediSearch OSS Redis 7.4 起 BSD
notes: |
  Redis docs GitHub: redis/docs (可未来 git-clone). 本快照仅抓 vectors 主 page + 公知补充.
  RediSearch 在 Redis 8 起 included as default in OSS Redis (BSD), 不再需要 Redis Stack 单独安装.
---

# Redis docs snapshot

- `vectors.md` — vector search 主 page, HNSW/FLAT + BM25 hybrid + in-memory architecture
