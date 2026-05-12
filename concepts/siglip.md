---
title: SigLIP（Sigmoid Loss for Language-Image Pre-training）
type: concept
sources: [zhai-2023-siglip, radford-2021-clip]
related: [clip.md, matryoshka-embedding.md, ../topics/multimodal-embedding-retrieval.md]
created: 2026-05-12
updated: 2026-05-12
---

# SigLIP (Sigmoid Loss for Language-Image Pre-training)

**TL;DR**: Google DeepMind Zhai et al. 2023 ICCV [zhai-2023-siglip] 提出 **CLIP 替代 contrastive loss**——用 **pairwise sigmoid loss** 替代 CLIP 的 softmax-based InfoNCE. **对 wiki 内 vector DBs 的核心价值**: (1) **CLIP 的现代默认替代**——SigLIP 在 production multimodal embedding deployment 中 surpassing CLIP, 是 wiki 内 multimodal embedding model selection 的当前主流; (2) **Sigmoid loss 不需要 global pairwise similarity normalization**——理论上让 batch size 进一步 scale up + 小 batch size 上更好; (3) **84.5% ImageNet zero-shot accuracy** with locked-image tuning + 4 TPUv4 chips × 2 days——production-feasible training cost (CLIP 原 paper 用 256 V100 × 12 days); (4) **Batch size analysis insight**——sigmoid loss 在 batch<16k 显著优于 softmax, 32k batch is sufficient for image-text pretraining (vs CLIP 32K is minimum); (5) **SigLIP 2 / SigLIP-shape (2024+) 后续 variants** 进一步 improve, 已成为多模态 production 默认 baseline. **Production 现状**: HuggingFace `google/siglip-base-patch16-224` / `google/siglip-large-patch16-384` 等 model 广泛部署; OpenAI / Cohere 等闭源 multimodal embedding 在算法层也参考 SigLIP. [zhai-2023-siglip §1-4]

## 与 CLIP 关键差异

[per zhai-2023-siglip §3]

**CLIP InfoNCE loss** (softmax-based):
```
loss = softmax across N×N similarity matrix
     ≈ - log [ exp(sim(I_i, T_i)) / Σ_j exp(sim(I_i, T_j)) ]
```
- 需要 global pairwise similarity normalization (across all N samples in batch)
- 大 batch size 极重要 (paper 32K batch)
- Memory + compute scale O(N²)

**SigLIP sigmoid loss**:
```
loss = - log σ(z_ij)        for positive pair (i==j)
     - log σ(-z_ij)        for negative pair (i≠j)
where z_ij = t × sim(I_i, T_j) + b   # learnable temperature + bias
```
- 每 pair 独立 sigmoid loss (no global normalization)
- 不需要 global view, **可分布式 efficient**
- 小 batch 上效果显著优于 softmax

→ **SigLIP 是 CLIP 训练范式的 step change**: 从 "global softmax 大 batch" 到 "pairwise sigmoid 灵活 batch size".

## 关键性质

### Batch size scaling

[per zhai-2023-siglip §4]

- Sigmoid loss 在 batch=1k 仍可 work (vs softmax 需 ≥4k 才稳定)
- Sigmoid loss 在 batch=8k 已显著优于 softmax
- Sigmoid + softmax 在 batch≥32k 接近 (gap closes)
- **Practical implication**: SigLIP 让 **smaller team / fewer GPU** 训出 production-grade multimodal embedding

### Locked-image Tuning + 4 TPUv4 chips × 2 days = 84.5% ImageNet zero-shot

[per zhai-2023-siglip §4]

- Lock pre-trained image encoder (ImageNet-21K supervised)
- Train **only text encoder** via SigLIP loss
- 4 TPUv4 × 2 days achieves **84.5% ImageNet zero-shot**——vs CLIP 256 V100 × 12 days 76.2%
- **Production training cost 数量级降低**

### Production model variants

[per HuggingFace google/siglip-*]

- `google/siglip-base-patch16-224` (768-d output)
- `google/siglip-base-patch16-256` (768-d)
- `google/siglip-large-patch16-256` (1024-d)
- `google/siglip-large-patch16-384` (1024-d, paper "best")
- `google/siglip-so400m-patch14-384` (1152-d, scaled variant)
- SigLIP 2 (2024) 进一步 improve, 多 modalities + 更 SOTA

## 与 wiki 内 CLIP / 其他 multimodal embedding 关系

[per zhai-2023-siglip + concepts/clip.md]

| | CLIP (Radford 2021) | **SigLIP (Zhai 2023)** |
|---|---|---|
| Loss | InfoNCE softmax | **Pairwise sigmoid** |
| Batch normalization | global N×N softmax | per-pair sigmoid |
| Min batch size | ~32K | ~1K (works), ~8K (优于 CLIP) |
| Training cost (84.5% ImageNet zero-shot equiv) | ~256 V100 × 12 days | **4 TPUv4 × 2 days** |
| Variants | RN50 / ViT-B/16, L/14, L/14@336 (512-1024d) | base / large / so400m (768-1152d) |
| Production deployment | 仍主流但 SigLIP rapidly displacing | **2024-2025 multimodal default** |
| 与 wiki vector DBs | All vendor cosine ANN over CLIP | **All vendor cosine ANN over SigLIP** (same infrastructure) |

→ Vector DB 端**完全 transparent**: CLIP-style embedding 与 SigLIP-style embedding 都是 L2-normalized + cosine ANN, vendor 直接服务两者.

## Open Questions

- **SigLIP vs CLIP 在 production cross-modal retrieval head-to-head benchmark**: vendor 端实测对比 不公开
- **SigLIP 2 / SigLIP-shape extensions production**: 2024+ variants 在 production case 不公开
- **SigLIP × MRL prefix-truncation**: SigLIP-MRL trained variant 不存在公开
- **SigLIP multimodal extension beyond image+text**: ImageBind 6-modality vs SigLIP image+text only
- **SigLIP quantization tolerance**: int8 / binary / PQ SigLIP recall 退化曲线 不公开

Cited by: 待 query 引用
