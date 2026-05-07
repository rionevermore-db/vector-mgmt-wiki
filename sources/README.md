# Sources

> Raw materials。**不可变**——文件一旦放入只能新增不能改。

## Source 清单

| Key | 标题 | 作者 | 年份 | 类型 | 文件 | Ingest 日期 |
|---|---|---|---|---|---|---|
| malkov-2016-hnsw | Efficient and Robust Approximate Nearest Neighbor Search using Hierarchical Navigable Small World Graphs | Yu. A. Malkov, D. A. Yashunin | 2016 (arXiv) / 2020 (IS) | paper | papers/malkov-2016-hnsw.pdf | 2026-04-30 |
| jegou-2011-pq | Product Quantization for Nearest Neighbor Search | Hervé Jégou, Matthijs Douze, Cordelia Schmid | 2011 (TPAMI) | paper | papers/jegou-2011-pq.pdf | 2026-05-07 |
| fu-2017-nsg | Fast Approximate Nearest Neighbor Search With The Navigating Spreading-out Graph | Cong Fu, Chao Xiang, Changxu Wang, Deng Cai | 2017 (arXiv) / 2019 (VLDB) | paper | papers/fu-2017-nsg.pdf | 2026-05-07 |
| guo-2019-scann | Accelerating Large-Scale Inference with Anisotropic Vector Quantization | Ruiqi Guo, Philip Sun, Erik Lindgren, Quan Geng, David Simcha, Felix Chern, Sanjiv Kumar | 2019 (arXiv) / 2020 (ICML) | paper | papers/guo-2019-scann.pdf | 2026-05-07 |
| johnson-2017-faiss-gpu | Billion-scale similarity search with GPUs | Jeff Johnson, Matthijs Douze, Hervé Jégou | 2017 (arXiv) / 2021 (IEEE TBD) | paper | papers/johnson-2017-faiss-gpu.pdf | 2026-05-07 |
| douze-2024-faiss-library | The Faiss Library | Matthijs Douze, Alexandr Guzhva, Chengqi Deng, Jeff Johnson, Gergely Szilvasy, Pierre-Emmanuel Mazaré, Maria Lomeli, Lucas Hosseini, Hervé Jégou | 2024 (arXiv) | paper | papers/douze-2024-faiss-library.pdf | 2026-05-07 |
| subramanya-2019-diskann | DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node | Suhas Jayaram Subramanya, Devvrit, Rohan Kadekodi, Ravishankar Krishnaswamy, Harsha Vardhan Simhadri | 2019 (NeurIPS) | paper | papers/subramanya-2019-diskann.pdf | 2026-05-07 |
| chen-2021-spann | SPANN: Highly-efficient Billion-scale Approximate Nearest Neighbor Search | Qi Chen, Bing Zhao, Haidong Wang, Mingqin Li, Chuanjie Liu, Zengzhong Li, Mao Yang, Jingdong Wang | 2021 (NeurIPS) | paper | papers/chen-2021-spann.pdf | 2026-05-07 |
| wang-2021-milvus | Milvus: A Purpose-Built Vector Data Management System | Jianguo Wang, Xiaomeng Yi, Rentong Guo, Hai Jin, Peng Xu, Shengjun Li, Xiangyu Wang, Xiangzhou Guo, Chengming Li, Xiaohai Xu, Kun Yu, Yuxing Yuan, Yinghao Zou, Jiquan Long, Yudong Cai, Zhenxiang Li, Zhifeng Zhang, Yihua Mo, Jun Gu, Ruiyi Jiang, Yi Wei, Charles Xie | 2021 (SIGMOD) | paper | papers/wang-2021-milvus.pdf | 2026-05-07 |
| milvus-docs | Milvus 官方文档（v2.6.x） | Zilliz / Milvus community | 2026 (commit f7f7c21) | docs | docs/milvus/ | 2026-05-07 |
| xu-2023-spfresh | SPFresh: Incremental In-Place Update for Billion-Scale Vector Search | Yuming Xu, Hengyu Liang, Jin Li, Shuotao Xu, Qi Chen, Qianxi Zhang, Cheng Li, Ziyue Yang, Fan Yang, Yuqing Yang, Peng Cheng, Mao Yang | 2023 (SOSP) | paper | papers/xu-2023-spfresh.pdf | 2026-05-07 |

## 命名约定

- 论文：`papers/<first-author-year-shortname>.pdf`，例：`chen-2021-spann.pdf`
- 文档：`docs/<vendor>-<topic>.md`，例：`pinecone-sharding.md`

## 引用方式

在 wiki page 的 frontmatter `sources` 字段里写本表的 Key（如 `[chen-2021-spann]`）。
