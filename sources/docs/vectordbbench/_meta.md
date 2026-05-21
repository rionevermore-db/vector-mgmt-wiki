---
source-url: https://github.com/zilliztech/VectorDBBench
upstream-leaderboard: https://zilliz.com/vdbbench-leaderboard
fetched-at: 2026-05-21
acquisition-method: webfetch-summary
files: [vectordbbench.md]
---

# VectorDBBench (VDBBench) 文档快照

Zilliz 维护的 **system-level** vector DB benchmark 工具 + 公开 leaderboard。

## 获取流程

1. WebFetch `https://raw.githubusercontent.com/zilliztech/VectorDBBench/main/README.md`（methodology + 30+ client 列表 + 4 case 类型）
2. WebSearch + VDBBench 1.0 发布说明（Milvus/Zilliz blog）补 streaming insertion-under-load 细节（Cohere-10M, 500 rows/s）
3. Leaderboard（`zilliz.com/vdbbench-leaderboard`）为 JS 动态页，**未抓到逐条数字**——具体 QPS/QP$ 数值随版本与提交滚动变化,引用时应指向 live leaderboard 而非冻结某次数

## 引用约定

- Citation 形如：`[per sources/docs/vectordbbench/vectordbbench.md]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `vectordbbench-docs`
- **数字易变**：leaderboard 结果随版本滚动；引用方法论稳定,引用绝对数值须注明 "as of" 与不可冻结
