---
title: CLIP（Contrastive Language-Image Pre-training）
type: concept
sources: [radford-2021-clip, kusupati-2022-matryoshka, formal-2021-splade-v2]
related: [../systems/vespa.md, product-quantization.md, scann.md, hnsw.md, matryoshka-embedding.md, splade-sparse-retrieval.md, ../topics/multimodal-embedding-retrieval.md, ../topics/multi-vector-queries.md, ../topics/mips-vs-l2-nn.md, ../topics/adaptive-retrieval-shortlist-rerank.md, ../topics/sparse-dense-hybrid-retrieval.md]
created: 2026-05-11
updated: 2026-05-12 (SPLADE as sparse-side neighbor in hybrid retrieval)
---

# CLIP

**TL;DR**: OpenAI 2021 ICML 论文 [radford-2021-clip] 提出的**多模态 dual-encoder contrastive embedding model**——首次证明 web-scale (400M image-text pair) 自监督训练的 image/text 双塔模型能在 **30+ 个下游 vision benchmark 上 zero-shot 击败 fully supervised baseline**。**对 wiki 内 vector DBs 的核心贡献**：(1) **joint multi-modal embedding space**——image 与 text 共享同一 512/1024-d 向量空间，所有跨模态检索 (text→image / image→text) 变成**单一 cosine ANN query**；(2) **L2-normalized + cosine similarity + learnable temperature τ**——直接 fit wiki 内所有 vector DBMS 的 cosine ANN 主要 distance metric (Milvus/Pinecone/Qdrant/Weaviate/Vespa/Turbopuffer 全部 first-class)；(3) **Linear projection only** from each encoder representation——避免 non-linear projection 带来的训练复杂度，保持 embedding space 几何性质对 ANN-friendly；(4) **Foundational baseline for entire multimodal retrieval frontier**——SigLIP / ALIGN / ImageBind 等后续多模态 embedding 全部以 CLIP 为对照。CLIP 不解决 vector DB 自身设计问题，但**奠定 vector DBs 服务多模态 production workload 的算法基础**。[radford-2021-clip §2-3]

## 提出背景

[per radford-2021-clip §1]

CV 之前的 SOTA 范式 (2015-2020)：
- Pre-train on ImageNet 1000 类 → fine-tune to downstream
- 严重 dataset-specific overfitting + 限制 categorical breadth (1000 类硬上限)
- 添加新 visual concept 需要 labeled data + retrain

NLP 同时期 (GPT / BERT / T5) 走出"task-agnostic + scaled web data + zero-shot"路径——CLIP **把 NLP 这套范式搬到 CV**：
- 不预测 ImageNet 1000 类, 而是**预测 "image 和 text 是否成对"**
- 400M (image, text) pairs from web → 任意 visual concept via natural language reference
- Zero-shot transfer to 30+ downstream tasks without retraining

**核心 insight**：natural language 作为 supervision signal 比 crowd-labeled categories 灵活——既能表达 visual concept, 又是 vector DB 自然 query 接口。

## 关键性质

### 1. 双塔架构 + 共享 embedding space

[per radford-2021-clip §2.4, Figure 3 pseudocode]

```python
# Image encoder: ResNet or ViT → I_f (image features)
# Text encoder: Transformer → T_f (text features)
# Linear projection only (not non-linear):
I_e = l2_normalize(np.dot(I_f, W_i), axis=1)  # image embedding
T_e = l2_normalize(np.dot(T_f, W_t), axis=1)  # text embedding

# Joint multi-modal embedding space:
# I_e, T_e ∈ R^d (d = 512 for base, 768 for large)
# L2-normalized → cosine similarity = inner product

# Contrastive objective (symmetric InfoNCE):
logits = np.dot(I_e, T_e.T) * np.exp(t)  # [n, n], scaled by learnable τ
labels = np.arange(n)                     # diagonal pairs are positive
loss_i = cross_entropy(logits, labels, axis=0)   # image→text
loss_t = cross_entropy(logits, labels, axis=1)   # text→image
loss = (loss_i + loss_t) / 2
```

**关键 design 决策**：
- **Linear projection** (`np.dot(I_f, W_i)`) 而非 non-linear——CLIP 论文 §2.4 明示 "We did not notice a difference in training efficiency between the two versions and speculate that non-linear projections may be co-adapted with details of current image only in self-supervised representation learning methods"
- **L2 normalization** 让 cosine similarity = inner product → vector DB 实现 trivial
- **Learnable temperature τ** clipped to ≤100, 直接优化 log-parameterized scalar——training stability 关键

### 2. 训练规模 + WIT 数据集

[per radford-2021-clip §2.2, §2.5]

| 维度 | 配置 |
|---|---|
| Dataset | **WIT** (WebImageText) — 400M (image, text) pairs from web |
| Construction | 500K base queries (English Wikipedia words occurring ≥100 times + bigrams + WordNet synsets), ~20K pairs/query, total similar word count to GPT-2 WebText |
| Batch size | **32,768** (extremely large for N² contrastive) |
| Optimizer | Adam + decoupled weight decay + cosine LR schedule |
| Training time | RN50×64: **18 days × 592 V100 GPUs**; ViT-L/14: **12 days × 256 V100 GPUs** |
| Mixed precision | yes (memory + speed) |
| Gradient checkpointing | yes |
| Half-precision Adam stats | yes (memory) |
| Embedding similarity sharding | local batch pairwise similarity computed by each GPU |
| Temperature τ init | 0.07 |
| ViT-L/14 extra | +1 epoch at 336 px resolution → ViT-L/14@336px (paper's "best" model) |

### 3. 模型变体与 embedding 维度

[per radford-2021-clip §2.4]

5 个 ResNet + 3 个 ViT 变体：

| Architecture | Parameters | Embedding dim |
|---|---|---|
| ResNet-50 (RN50) | ~25M (Image) + 63M (Text) | 1024 |
| ResNet-101 | ~45M + 63M | 512 |
| RN50×4 / RN50×16 / RN50×64 (EfficientNet-style scaling) | scale by 4×/16×/64× compute | 640 / 768 / 1024 |
| ViT-B/32 | 87M + 63M | 512 |
| ViT-B/16 | 87M + 63M | 512 |
| ViT-L/14 / ViT-L/14@336px | 307M + 63M | **768** |

→ Vector DB 常见生产 deployment 选 **ViT-L/14 (768-d) 或 ViT-B/32 (512-d)**——前者质量上限, 后者 throughput friendly.

### 4. Zero-shot transfer 机制

[per radford-2021-clip §3.1]

```
1. 对每 dataset 的 N 个 class names 写成 prompt: "A photo of a {class}"
2. Text encoder 把 N 个 prompt 编码为 N 个 T_e_class
3. Image encoder 把 query image 编码为 I_e_query
4. 计算 N 个 cosine similarities = I_e_query · T_e_class.T (× τ scaled, softmax)
5. argmax → predicted class
```

→ 这正是 **vector DB 多模态 retrieval 的核心 query pattern**：
- Text → image 检索：`text encoder(query) → cosine ANN over image embedding namespace`
- Image → image 检索：`image encoder(query_image) → cosine ANN over image embedding namespace`
- Text → text 检索：`text encoder(query) → cosine ANN over text embedding namespace`
- **同一 embedding space** → 三种 retrieval 共用一套 vector DB index

### 5. Prompt engineering + ensembling

[per radford-2021-clip §3.1.4, Figure 4]

- "A photo of a {label}" 比 raw "{label}" 提升 ImageNet 1.3%
- 80 个 ensemble prompts (e.g., "A photo of a big {label}", "A photo of a small {label}") 在 embedding space 平均 → 单一 caching-friendly classifier
- Prompt engineering + ensembling 共 +5% on ImageNet

→ **Vector DB workload 含义**：production CLIP retrieval 系统 cache 1 个 averaged text embedding per query class, 跑 ANN against image embedding store. Vector DB 端不变, embedding side 工程化.

## 与同类对比

| | **CLIP (2021)** | Visual N-Grams (2017) | BERT / SBERT (text-only) | ResNet pre-train (ImageNet 21K) |
|---|---|---|---|---|
| Modality | image + text dual-encoder | image-only with text supervision | text-only | image-only |
| Dataset scale | **400M pairs** | YFCC100M filtered (~10M) | C4 / Wikipedia text | ImageNet 21K (~14M labeled images) |
| Objective | symmetric contrastive InfoNCE | bag-of-n-grams prediction | masked language modeling (text) | softmax cross-entropy classification |
| Embedding space | **joint image+text shared** | n-gram bag-of-words feature | text-only | image-only feature |
| ImageNet zero-shot | **76.2%** (ViT-L/14@336) | 11.5% | n/a | n/a |
| Vector DB direct use | **cosine ANN over joint space** | impractical (BoW) | text retrieval only | image classification (no retrieval native) |
| Prompt engineering | "A photo of a {label}" | n/a | requires task-specific head | requires fine-tune |
| Cross-modal retrieval | **first-class** | impractical | needs separate image encoder | needs separate text encoder |
| Inspired后续 | SigLIP, ALIGN, ImageBind, OpenCLIP | (deprecated path) | SimCSE, GTR, BGE | DINO, MAE, MoCo |

→ **CLIP 是 first major model** 把 "natural language as supervision" 与 "contrastive joint embedding" 结合到 image domain, **奠定 vector DB 服务多模态 production 的算法基础**.

## 典型实现 / Vector DB 集成 pattern

### Pattern 1: 单一 vector field 多模态共存

[per [Vespa](../systems/vespa.md) tensor framework + CLIP cross-modal mechanic]

```
# Vespa schema (per sources/docs/vespa)
schema product {
  field image_url type string { ... }
  field caption type string { ... }
  field clip_embedding type tensor<float>(x[768]) {     # ViT-L/14 output
    indexing: attribute | index
    attribute { distance-metric: angular }              # cosine
    index { hnsw { max-links-per-node: 16, neighbors-to-explore: 200 } }
  }
}

# Query: text→image
"select * from product where
   {targetHits: 100} nearestNeighbor(clip_embedding,
     vespa-embedded-clip-text-encoder(['italian leather wallet']))"

# Query: image→image
"... nearestNeighbor(clip_embedding,
     vespa-embedded-clip-image-encoder([query_image_bytes]))"
```

**所有 5 OSS vector DBMS + 2 closed SaaS** ([Milvus](../systems/milvus.md) / [Qdrant](../systems/qdrant.md) / [Weaviate](../systems/weaviate.md) / [Vespa](../systems/vespa.md) / [Pinecone](../systems/pinecone.md) / [Turbopuffer](../systems/turbopuffer.md) + [Faiss](../systems/faiss.md)) 都能 native 存储 CLIP embedding + cosine ANN——CLIP 是 wiki 内 vector DB 共通支持的 multimodal 算法.

### Pattern 2: 多 vector field separation

[per [Milvus](../systems/milvus.md) multi-vector field v2.6+ / Weaviate named vectors]

```
# Milvus schema
collection product {
  field image_embedding: float_vector(dim=768)     # CLIP image
  field text_embedding: float_vector(dim=768)      # CLIP text (caption / metadata)
  field hybrid_score: composite                    # weighted multi-vector query
}
```

→ 单 doc 存 image + text 两个 CLIP embedding (different content for each)——多 vector query 可 fuse cross-modal retrieval signal.

### Pattern 3: CLIP 作为 sparse-dense hybrid 的 dense path

[per [Vespa hybrid](../systems/vespa.md) + [Weaviate hybrid](../systems/weaviate.md)]

Web search / e-commerce production retrieval 常用 (1) BM25 over product description text + (2) CLIP cross-modal vector retrieval + (3) 简单线性融合 → 单结果集. Vector DB 端只需 cosine ANN + BM25 inverted index 并存.

## CLIP-style Embedding 在 vector DB 上的 production scale 数据点

[per wiki 内 vendor production case]

- **[Vespa](../systems/vespa.md)**: tensor framework + ONNX 直接跑 CLIP encoder; production multimodal retrieval 主流
- **[Pinecone](../systems/pinecone.md)**: Pinecone Inference 内置 CLIP-style embedding 调用
- **[Turbopuffer](../systems/turbopuffer.md)**: "Adding additional multi-modal data to query, e.g. embeddings of the images (Cohere image model, Voyage image model)" — docs 显式提及 multimodal extension
- **[Milvus](../systems/milvus.md)**: 多 vector field + DiskANN/HNSW/CAGRA index 服务 CLIP embedding

## CLIP Embedding 的关键限制（vector DB workload 相关）

[per radford-2021-clip §6 Limitations]

1. **Out-of-distribution failure**: MNIST 仅 88% (raw pixel logistic regression 超过). **Vector DB 含义**: production CLIP retrieval system 必须 monitor OOD detection + fallback path 对 unseen 数据 distribution
2. **Fine-grained classification weak**: Stanford Cars, FGVC Aircraft 差 ResNet-50 features 10%+. **Vector DB 含义**: 细分领域 (车型 / 飞机型号) 不能依赖 CLIP zero-shot, 需 domain-specific fine-tune (CLIP fine-tune 路径 OpenCLIP / LAION-CLIP variants)
3. **Specialized tasks**: counting / abstract concepts / 卫星图像 / 医学影像 / 距离估计 near-chance. **Vector DB 含义**: medical imaging / satellite vector DB 需 domain-specific multimodal model (BiomedCLIP / RemoteCLIP 等)
4. **Class design 影响**: prompt 选择 / class names 严重影响 output (FairFace bias). **Vector DB 含义**: production system 需 audit class label / query template; "A photo of a {label}" 是 default 但 domain-specific prompt 更好
5. **不可生成**: CLIP 限于 retrieval (从已知 set 选一个) 而非 caption generation. **Vector DB 含义**: 不能用 CLIP 替代 LLM caption / image generation; CLIP 是 retrieval-only
6. **Few-shot 反直觉**: zero-shot 直接用 prompt 反而比 1-shot / 4-shot linear probe 还好 (§3.1.5 Figure 6). **Vector DB 含义**: prompt engineering 比"加几个 labeled example"更经济
7. **Social bias**: race / gender / age classification (FairFace Table 5-7) — black images 14% mis-classified into "non-human" vs <8% other races; 16.5% males 9.8% females mis-classified as crime. **Vector DB 含义**: production retrieval 需 bias mitigation + content moderation layer

## 后续演化与现代多模态 embedding model 生态

CLIP 启动了 modern multimodal embedding model 整个 frontier:

- **OpenCLIP (LAION 2022+)**: 开源 reproduction + 大规模 (LAION-2B / LAION-5B)
- **ALIGN (Google 2021)**: 类似 CLIP, 噪声大数据集 1.8B pairs
- **SigLIP (Google 2023)**: sigmoid loss 替换 softmax InfoNCE — 计算更便宜 + 一般质量更优
- **EVA-CLIP (BAAI 2022+)**: 更大 model scaling
- **ImageBind (Meta 2023)**: 6 modality joint space (image / text / audio / depth / thermal / IMU)
- **DFN / SigLIP-2 (Google 2024+)**: data filtering + 改进训练
- **Cohere multilingual CLIP / Voyage multimodal** (commercial): 多语言 + multimodal embedding production

→ Vector DB 端**算法基本不变** (all 用 cosine ANN over normalized embeddings)——演化主要在 embedding model side, vector DB 自动 inherit benefits via model swap (per [Turbopuffer namespace-as-tenant](../systems/turbopuffer.md) or [Vespa application package](../systems/vespa.md) atomic deploy patterns).

## Open Questions

- **CLIP-trained 后续 fine-tune 的 embedding 几何变化**: pre-train 的 cosine geometry 是否在 fine-tune 后 preserve? Vector DB index (HNSW α / SPFresh centroid placement) 在 embedding distribution shift 下是否仍 optimal? wiki / 论文都不涵盖
- **CLIP vs SigLIP retrieval head-to-head**: paper-level 实测对比 vector DB-end recall / QPS / cost? 不存在公开数据
- **CLIP embedding quantization tolerance**: 768-d float32 vs int8 vs RaBitQ vs binary 各 quantizer 下 cross-modal recall 退化曲线? wiki zero coverage
- **Multimodal retrieval 标准 benchmark**: 类似 MTEB-image 的 ANN-friendly 多模态 retrieval benchmark? 现有 CLIP benchmark (Flickr30K / MS-COCO / ImageNet zero-shot) 都不是 vector-DB-aware
- **Cross-domain CLIP variant routing**: production system 需要 in-domain (BiomedCLIP for medical) vs general CLIP routing logic; vector DB schema 如何支持? 单 namespace 单 model 还是 multi-namespace 多 model fusion?
- **Temperature τ 在 vector DB 端的应用**: CLIP τ 是 training-time scalar; vector DB query-time 通常不暴露 τ, 但 logit scaling 影响相似度 percentile 解释——Pinecone / Vespa rerank 阶段使用 τ-scaled cosine? docs 不细谈
- ~~**Embedding versioning + multi-CLIP-version coexistence**: ViT-B/32 vs ViT-L/14 vs ViT-L/14@336px 三 model 输出 dim 不同 (512 vs 768)——同一 vector DB 跨 model version migration 路径 (per [embedding-update query](../evolution/tracking-queries.md))~~ **2026-05-11 ingest [kusupati-2022-matryoshka] 已部分解**: [Matryoshka Representation Learning](./matryoshka-embedding.md) (MRL) 用 ONE model 训练时显式优化 O(log d) 个 nested prefix——把 "多 variant 不同 dim 各自独立训练" 模式**取代为 "单 model 多 prefix dim"** 模式. **production 影响**: OpenAI text-embedding-3 / Cohere embed-v4 / Voyage-3 / Qwen3-VL-Embedding-8B 等当前 production embedding 全部 MRL-trained——客户在 vector DB 端可 (a) store 全维一次, (b) query 时按 latency budget 选 prefix dim. CLIP-style 多 variant 矩阵 (RN50 1024-d, RN101 512-d, ViT-B/32 512-d, ViT-L/14 768-d 各自独立训练) 是 pre-MRL 范式; post-MRL production 应用倾向**单一 MRL-trained model 跨 prefix dim**——这是 CLIP 之后 embedding-side 最大演化. **仍 open**: CLIP 自身 (2021) 不是 MRL-trained——后续 OpenCLIP / SigLIP variants 是否引入 MRL training? OpenCLIP-MRL / SigLIP-MRL 不存在公开. Vespa "matryoshka tensor" cell type 与 CLIP 直接 fit (per [systems/vespa.md](../systems/vespa.md))。
- **CLIP-style joint space + spatial (geo) 三模融合**: 论文不涵盖, 但 production system (Bing / Google) 同时支持 vector + scalar + geo + 是否有 unified joint embedding training? Wiki 内 zero coverage
- **Few-shot CLIP + zero-shot CLIP fusion**: paper Figure 6 zero-shot 反而比 1-shot linear probe 好——production retrieval 应该如何使用 user feedback (clicks / dwell time) 改进 CLIP-based ranking? wiki / 论文都不涵盖

Cited by: 待 query 引用
