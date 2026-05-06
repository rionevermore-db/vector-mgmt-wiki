# CLAUDE.md — Vector Management Wiki Schema

## 仓库目的

本仓库是一个 LLM 维护的、以 **vector management for LLM inference** 为主题的研究知识库。
模式参考 Andrej Karpathy 的 LLM-Wiki gist：
<https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f>

核心思想：**LLM 不是问答机器，而是知识簿记员**——把人读源材料的认知劳动沉淀为可互链、可检索、可演化的 markdown 知识库。

## 三层架构

```
┌─────────────────────────────────────────────┐
│  Layer 3: Schema (this file: CLAUDE.md)     │  ← 规则、契约
├─────────────────────────────────────────────┤
│  Layer 2: Wiki (concepts/ systems/ ...)     │  ← LLM 生成、可重建
├─────────────────────────────────────────────┤
│  Layer 1: Sources (sources/papers, docs)    │  ← 不可变的真相
└─────────────────────────────────────────────┘
```

- **Layer 1（Sources）**：raw materials，不可变。每篇论文、每份产品文档进来就放这里，永不修改。
- **Layer 2（Wiki）**：LLM 根据 sources 生成的派生内容，可以随时重建。
- **Layer 3（Schema）**：本文件。定义 layer 2 长什么样、怎么演化。

## 目录结构

```
vector-mgmt-wiki/
├── CLAUDE.md           ← 本文件（schema）
├── README.md           ← 访客入口（人类读）
├── index.md            ← 全 wiki 目录，按主题分类
├── log.md              ← append-only 操作日志
├── sources/
│   ├── papers/         ← 论文 PDF / markdown 复刻
│   ├── docs/           ← 产品文档、engineering blog
│   └── README.md       ← source 清单（标题、作者、URL、ingest 日期）
├── concepts/           ← 算法 / 数据结构 page
├── systems/            ← 系统 / 产品 page
├── topics/             ← 跨概念主题 page
├── benchmarks/         ← 测评结果 + 方法论
└── queries/            ← 高价值 query 答案存档
```

## 五种 Page Type 与模板

每个 page 必须以 frontmatter 开头：

```yaml
---
title: <人类可读标题>
type: concept | system | topic | benchmark | query
sources: [paper-key-1, doc-key-2]   # 引用 sources/README.md 中的 key
related: [page-1, page-2]            # 同类或相邻 page 的相对路径
created: 2026-04-29
updated: 2026-04-29
---
```

### `concepts/` — 算法 / 数据结构

```markdown
# <名称>

**TL;DR**: 一句话定义。

## 提出背景
谁、什么时候、为了解决什么问题。

## 关键性质
- 时间复杂度 / 空间复杂度
- 准确率 vs 速度的权衡点
- 适用规模

## 与同类对比
| | 本算法 | 类似算法 A | 类似算法 B |
|---|---|---|---|
| ... | | | |

## 典型实现
工程实现要点；引用 systems/ 中的相关 page。

## Open Questions
- 我还没搞懂的点
```

### `systems/` — 系统 / 产品

```markdown
# <系统名>

**TL;DR**: 一句话概括它是什么、解决什么场景。

## 架构图
ASCII 或链接到 sources/docs 中的图。

## 数据流 / 控制流

## 关键设计决策
逐条列举，每条注明 trade-off。

## Scale 边界
该系统能撑到多大？瓶颈在哪？

## 生产案例
谁在用、规模如何。

## Open Questions
```

### `topics/` — 跨概念主题

跨多个 concepts/ 和 systems/ 的横切面问题。例如"千亿规模分片权衡"、"GPU 加速 ANN 的现状"。

```markdown
# <主题>

**TL;DR**: 这个主题在问什么。

## 问题陈述
为什么这个问题重要、约束是什么。

## 相关概念
列举依赖的 concepts/ 和 systems/ page。

## 工业方案对比
| 方案 | 选择者 | 优势 | 劣势 |
|---|---|---|---|

## Open Questions
```

### `benchmarks/` — 测评

```markdown
# <benchmark 名>

**TL;DR**: 测什么、谁测的。

## 实验设置
- 数据集
- 硬件
- 指标

## 结果
表格 + 引用原始 source。

## 可信度评估
- 实验设计是否合理
- 是否有偏向
- 复现难度
```

### `queries/` — 高价值 query 答案存档

每次提出一个值得记录的复杂问题，把答案 + 引用归档于此。

```markdown
# <query 简短标题>

**Date**: 2026-04-29
**Question**: 原始问题（完整原文）

## TL;DR
2-3 句结论。

## Answer
完整答案。

## Cited Pages
- [page-1](../concepts/page-1.md)
- [page-2](../systems/page-2.md)

## Follow-up Questions
后续值得追问的点。
```

## 命名约定

- 所有文件名英文 kebab-case（如 `disk-ann.md`、`gpu-aware-ann.md`）
- `sources/papers/<first-author-year-shortname>.pdf`（如 `chen-2021-spann.pdf`）
- `sources/docs/<vendor>-<topic>.md`（如 `pinecone-sharding.md`）
- 中文标题写在 page 内的 frontmatter `title` 字段

## Cross-link 规则

1. **每次 ingest 必须更新至少 1 个现有 page 的 `related`** 字段（保证 wiki 不是孤岛）
2. **每次 query 答案必须列出 cited pages**，并双向更新——被引用的 page 也要在末尾加一行 "Cited by: queries/xxx"
3. **Page 中提到的概念**首次出现时用相对路径 link：`[HNSW](../concepts/hnsw.md)`

## Source 引用规则

- 每个 page 的 frontmatter `sources` 字段必填，至少 1 个 source key
- Page 正文中陈述事实/数字，**必须**就近标注 source（用脚注或行内引用）
- 没有 source 支持的内容用 `> [推测]` 引用块标记，不要混入正文

> **Why**：一个月后 LLM/你都分不清"这是论文写的"还是"我或 LLM 编的"。强制 citation 是 wiki 长期可信度的唯一保障。

## log.md 格式

每条记录一行：

```
2026-04-29 | ingest | sources/papers/chen-2021-spann.pdf | 新增 disk-ann.md / spann.md，更新 sharding.md disk-vs-mem.md
2026-04-30 | query  | giga-scale-sharding | 引用 6 个 page，存档于 queries/giga-scale-sharding.md
2026-04-30 | lint   | 全库扫描 | 标记 3 个 stale claim，1 个 orphan page
```

## 三种工作流

### Ingest（消化新源）

**入口路径有三种：**

- **路径 A（论文，标题驱动）**：用户给一个标题/作者+年份/主题描述（如 "ingest SPANN 论文"、"ingest Manu Milvus 2022"），LLM 自动定位 + 下载 PDF。
- **路径 B（论文，手动 fallback）**：用户自己下载 PDF 放到 `sources/papers/` 后告诉 LLM 文件路径。**适用场景**：付费墙论文（ACM/IEEE 闭门）、公司内部文档、扫描版老论文、用户已经手上有 PDF 不想重新下。
- **路径 C（产品文档，全量抓取）**：用户给产品名（如 "ingest Pinecone 文档"），LLM 把全套官方文档拉到 `sources/docs/<vendor>/`。

#### 路径 A 步骤（自动获取）

1. **解析标题** → WebSearch 找规范来源；优先级：arXiv > 作者主页/官方 PDF > 会议官网 > 其他镜像
2. **向用户确认**："找到 `<full title>`，`<arxiv-or-source-url>`，下载并 ingest 吗？" → 等用户点头
3. **下载到 sources/**：用 `Invoke-WebRequest` 下到 `sources/papers/<first-author-year-shortname>.pdf`，符合命名约定
4. **进入步骤 5**（读 + ingest 流程）

> **路径 A 的失败/降级**：
> - WebSearch 找不到合法 URL → 报告用户、转路径 B
> - PDF 是付费墙（DOI 跳转、需登录）→ 报告用户、转路径 B
> - 找到的 URL 存在多个版本（preprint vs 期刊版、v1 vs v3）→ 列出选项让用户挑

#### 路径 B 步骤（手动）

1. 用户告知 PDF 路径（已放在 `sources/papers/`）
2. 如果文件名不符合 `<first-author-year-shortname>.pdf` 约定，**首次 ingest 时**询问用户能否重命名（一旦重命名后即冻结）

#### 路径 C 步骤（产品文档，全量抓取）

获取优先级从高到低：

1. **`llms.txt` / `llms-full.txt`** — 厂商主动合并的 LLM 友好文档。先 WebFetch `<site>/llms-full.txt` 或 `<site>/llms.txt`。**Pinecone / Anthropic / OpenAI** 等已有，趋势扩散中。一次拉到位，最干净。
2. **GitHub 文档仓库** — 已知映射：
   - Milvus → `milvus-io/milvus-docs`
   - Qdrant → `qdrant/landing_page`（docs 子目录）
   - Weaviate → `weaviate/weaviate-io`
   - Vespa → `vespa-engine/documentation`
   `git clone` 到 `sources/docs/<vendor>/`，保留 commit hash 写进 frontmatter
3. **Sitemap 爬取** — 兜底。读 `<site>/sitemap.xml` 列出 URL → 报告页数让用户确认是否全抓 → 逐页 WebFetch 转 markdown

**存储约定**：
- 全部落到 `sources/docs/<vendor>/`（子目录，不平铺），原始路径结构尽量保留
- 在 `sources/docs/<vendor>/_meta.md` 写 frontmatter：`source-url`（站点根）、`fetched-at`（日期）、`acquisition-method`（llms-txt / git-clone / sitemap-crawl）、`commit-hash`（如 git clone）
- 在 `sources/README.md` 表格里 source key 用 `<vendor>-docs`（如 `pinecone-docs`、`milvus-docs`），文件列指向子目录

**版本演化**：文档变了不要改原文件，新建 `sources/docs/<vendor>-<YYYY-MM>/`（如 `milvus-2026-08/`）作为新快照，旧快照保留作历史。

**Wiki 层硬约束（避免退化为文档镜像）**：
- 一个产品文档全套抓进来后，wiki 层**不应**机械生成数十上百个 page
- 默认产出：**1 个 `systems/<vendor>.md` 主 page**（架构、关键设计、scale 边界、生产案例）
- 仅在产品有专属算法/独创概念时增产 0-2 个 `concepts/<product-specific>.md`（如 Milvus 的 Knowhere、Pinecone 的 pod-based sharding）
- raw docs 留在 sources/docs/ 作为 citation 锚点，**正文引用要细到具体页**（如 `[per sources/docs/milvus/architecture/overview.md]`）

> **Why**：Wiki 的价值是"沉淀过的认知"，不是"docs 的复制"。如果一篇官方文档已经写得很好，wiki page 应该指向它而不是复述。

#### 公共步骤（路径 A、B、C 汇合）

5. 阅读 source，向用户口头报告 takeaway，等待用户补充
6. 决定该 source 影响哪些 page type，候选包括：
   - 新建 1 个或多个 page（concepts / systems / topics / benchmarks）
   - 更新若干已有 page（添加新 finding、修订过时陈述、加 cross-link）
7. 执行所有文件操作
8. 更新 `index.md` 的对应章节
9. 更新被影响 page 的 frontmatter `updated` 字段
10. 在 `sources/README.md` 表格追加该 source（key、标题、作者、年份、文件、ingest 日期）
11. 在 `log.md` 末尾追加一条 ingest 记录
12. 报告"本次 ingest 影响 N 个文件，新增 X 个、更新 Y 个"

> **目标**：一篇有信息量的论文应该触发 5-15 个文件改动。如果只触发 1-2 个，要么 source 信息密度低，要么 wiki 结构不够细——后者是 lint 的事。

### Query（用 wiki 回答问题）

输入：用户提出一个问题。

步骤：
1. 检索 wiki 中相关 page（按 `index.md` 和 `topics/` 入手）
2. 综合答案，正文中**就近标注**引用了哪个 page（`[per disk-ann.md]` 或类似）
3. 如果答案包含 wiki 没覆盖的内容，明确标注 `> [推测，wiki 未覆盖]`
4. 询问用户："这个 query 是否值得归档进 `queries/`？"
5. 如归档，按 query page 模板创建文件，并在 `log.md` 追加 query 记录

### Lint（保鲜）

定期（建议每 ~10 次 ingest 后或用户主动请求）执行：

1. **Orphan check**：扫所有 page，找没有任何其他 page 引用的孤岛
2. **Stale check**：扫包含 `[推测]` 的 page，看是否已经有新 source 可以替换
3. **Contradiction check**：扫多个 page 对同一概念的描述，标出矛盾
4. **Cross-link gap**：找应该互链但没互链的 page 对（基于关键词重叠）
5. 报告结果，让用户决定如何处理
6. 在 `log.md` 追加 lint 记录

> **目标**：lint 是 wiki 长期不腐烂的关键。"速成"靠 ingest，"保持专家"靠 lint。

## 几条硬约束

1. **Sources 不可修改**——`sources/` 下文件一旦放入只能新增不能改。如果发现 source 错了，新建一份并在 log 注明
2. **不放公司内部材料**到 sources/——本仓库 GitHub public，所有 source 必须公开可获取
3. **不要为 LLM 自动生成的内容标"已验证"**——除非用户明确说"我读过这个 page，它对了"
4. **每个 page 都必须能独立读懂**——不能写"如上所述"、"前文提到"这种依赖阅读顺序的表达
5. **Ingest 时 LLM 主动提问**——遇到不确定的归类（"这个 page 算 concept 还是 topic？"）必须问用户而不是自己拍板

## 给 Claude Code 的元指令

- 用户说 **"ingest <论文标题/作者+年份/主题>"** → 执行 Ingest workflow 路径 A（自动找 URL → 确认 → 下载 PDF → ingest）
- 用户说 **"ingest <source-path>"**（已存在的本地 PDF 路径）→ 执行 Ingest workflow 路径 B
- 用户说 **"ingest <产品名> 文档"**（如 "ingest Pinecone 文档"）→ 执行 Ingest workflow 路径 C（llms.txt → GitHub → sitemap 顺序尝试）
- 用户提问任何技术问题 → 默认走 Query workflow（先用 wiki 找答案）
- 用户说 **"lint"** → 执行 Lint workflow
- 用户说 **"plan"** → 报告当前 wiki 状态：page 数、最近 ingest、可见的覆盖盲点
