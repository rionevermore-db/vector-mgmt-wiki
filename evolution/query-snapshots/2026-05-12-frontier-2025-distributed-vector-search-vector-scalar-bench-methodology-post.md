---
query-key: vector-scalar-bench-methodology
date: 2026-05-12
phase: post
ingest-context: frontier-2025-distributed-vector-search
wiki-pages-total: 88
cited-pages: [concepts/frontier-2025-distributed-vector-search.md, topics/attribute-filtering.md]
cited-count: 2
---

# Post-snapshot (frontier-2025-distributed-vector-search): vector-scalar-bench-methodology

## TL;DR

**重大 NEW** (PathFinder 直接对答): 之前 wiki `topics/attribute-filtering.md` 已有 5-strategy 框架, PathFinder 引入 **6 类工作 cost-based optimizer** 选 strategy. 关键 NEW benchmark methodology:
1. **DNF (disjunctive normal form)** workload 必须 included——多属性 conjunction+disjunction 是真实 production 场景, 之前 benchmark 多只测 single-attribute filter
2. **Filter 5 strategy 之间不是 binary 选项**——PathFinder optimizer 选 *index subset* + *plan*, fair benchmark 需 specify "optimizer 决策时间预算" (sub-ms 内必须决定否则吃掉 query budget)
3. **9.8× throughput at recall 0.95** 是新 baseline——之前 wiki filter benchmark page 缺这种端到端数字
4. **Index borrowing** (相关 attribute 间 predicate 合成) = wiki 内首个 attribute-correlation aware filter strategy

关键 NEW research direction: fair filter-ANNS benchmark methodology = (a) fixed embedding model (per Ingest #12) + (b) DNF workload (per PathFinder) + (c) cost-budget-aware optimizer comparison + (d) selectivity 分布扫描.

## Cited Pages

- [concepts/frontier-2025-distributed-vector-search.md](../../concepts/frontier-2025-distributed-vector-search.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
