# 经典论文 (Classic Papers)

## 一句话概述

广告技术领域的经典论文，涵盖 CTR 预估、出价优化、推荐系统、隐私计算等方向，是深入理解广告算法的必读材料。

---

## CTR/CVR 预估

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2014 | [Practical Lessons from Predicting Clicks on Ads at Facebook](https://research.facebook.com/publications/practical-lessons-from-predicting-clicks-on-ads-at-facebook/) | Facebook | GBDT+LR 组合模型 |
| 2016 | [Wide & Deep Learning for Recommender Systems](https://arxiv.org/abs/1606.07792) | Google | Wide&Deep 架构 |
| 2017 | [DeepFM: A Factorization-Machine based Neural Network](https://arxiv.org/abs/1703.04247) | Huawei | FM + Deep 端到端 |
| 2017 | [Deep & Cross Network for Ad Click Predictions](https://arxiv.org/abs/1708.05123) | Google | 显式交叉网络 DCN |
| 2018 | [DIN: Deep Interest Network for Click-Through Rate Prediction](https://arxiv.org/abs/1706.06978) | Alibaba | Attention 建模用户兴趣 |
| 2019 | [DIEN: Deep Interest Evolution Network](https://arxiv.org/abs/1809.03672) | Alibaba | 兴趣演化建模 |
| 2018 | [ESMM: Entire Space Multi-Task Model](https://arxiv.org/abs/1804.07931) | Alibaba | CVR 样本选择偏差 |
| 2018 | [MMOE: Modeling Task Relationships in Multi-task Learning](https://dl.acm.org/doi/10.1145/3219819.3220007) | Google | 多任务学习 |
| 2020 | [PLE: Progressive Layered Extraction](https://dl.acm.org/doi/10.1145/3383313.3412236) | Tencent | 改进多任务学习 |
| 2019 | [DSIN: Deep Session Interest Network](https://arxiv.org/abs/1905.06482) | Alibaba | Session 级兴趣建模 |
| 2019 | [FiBiNET: Feature Importance and Bilinear feature Interaction](https://arxiv.org/abs/1905.09433) | Sina | 特征重要性 + 双线性交互 |
| 2021 | [DCN V2: Improved Deep & Cross Network](https://arxiv.org/abs/2008.13535) | Google | DCN 改进版 |
| 2020 | [AutoInt: Automatic Feature Interaction Learning](https://arxiv.org/abs/1810.11921) | Peking Univ | 自动特征交互 |

---

## 出价与预算优化

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2017 | [Bid Optimizing and Inventory Scoring in Targeted Online Advertising](https://dl.acm.org/doi/10.1145/2339530.2339655) | Yahoo | 出价优化理论 |
| 2018 | [Budget Constrained Bidding by Model-free Reinforcement Learning](https://arxiv.org/abs/1802.08365) | Alibaba | RL 出价 |
| 2018 | [Real-Time Bidding with Multi-Agent Reinforcement Learning](https://arxiv.org/abs/1802.09756) | UCL | 多智能体 RTB |
| 2019 | [Bid Optimization by Multivariable Control in Display Advertising](https://arxiv.org/abs/1905.10928) | Alibaba | PID 控制出价 |
| 2020 | [OCPC: A Bidding Strategy for Sponsored Search](https://dl.acm.org/doi/10.1145/3394486.3403229) | Baidu | oCPC 两阶段 |
| 2021 | [Auto-bidding in Real-Time Bidding](https://arxiv.org/abs/2106.07233) | Alibaba | 自动出价框架 |

---

## 推荐与广告融合

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2020 | [Ads Allocation in Feed via Constrained Optimization](https://dl.acm.org/doi/10.1145/3394486.3403391) | Meta | 信息流广告分配 |
| 2019 | [Deep Reinforcement Learning for Ad Allocation in Feed](https://arxiv.org/abs/1909.03602) | Alibaba | RL 广告插入 |
| 2021 | [Jointly Optimizing Revenue and User Experience](https://arxiv.org/abs/2106.01224) | ByteDance | 收入与体验平衡 |

---

## 搜索广告

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2013 | [DSSM: Learning Deep Structured Semantic Models](https://www.microsoft.com/en-us/research/publication/learning-deep-structured-semantic-models-for-web-search-using-clickthrough-data/) | Microsoft | 语义匹配模型 |
| 2007 | [Predicting Clicks: Estimating the CTR of New Ads](https://dl.acm.org/doi/10.1145/1242572.1242643) | Microsoft | 搜索广告 CTR 预估 |
| 2015 | [A Sub-linear, Massive-scale Look-alike Audience Extension](https://arxiv.org/abs/1503.02007) | Yahoo | Lookalike 扩量 |

---

## 竞价机制

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2007 | [Internet Advertising and the Generalized Second-Price Auction](https://www.aeaweb.org/articles?id=10.1257/aer.97.1.242) | Google/Yahoo | GSP 竞价理论 |
| 2019 | [Reserve Price Optimization at Scale](https://research.facebook.com/) | Facebook | 底价优化 |
| 2021 | [From GSP to First-Price Auction](https://arxiv.org/abs/2103.07730) | — | 一价竞价迁移 |

---

## 隐私与联邦学习

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2017 | [Federated Learning: Strategies for Improving Communication Efficiency](https://arxiv.org/abs/1610.05492) | Google | 联邦学习基础 |
| 2019 | [Federated Learning for Mobile Keyboard Prediction](https://arxiv.org/abs/1811.03604) | Google | 联邦学习应用 |
| 2006 | [Calibrating Noise to Sensitivity in Private Data Analysis](https://link.springer.com/chapter/10.1007/11681878_14) | — | 差分隐私基础 |
| 2014 | [RAPPOR: Randomized Aggregatable Privacy-Preserving Ordinal Response](https://arxiv.org/abs/1407.6981) | Google | 本地差分隐私 |

---

## 反作弊

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2012 | [Click Fraud Detection](https://dl.acm.org/doi/10.1145/2339530.2339547) | Microsoft | 点击欺诈检测 |
| 2020 | [A Survey on Click Fraud in Online Advertising](https://arxiv.org/abs/2008.12686) | — | 点击欺诈综述 |

---

## 在线学习

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2013 | [Ad Click Prediction: a View from the Trenches](https://dl.acm.org/doi/10.1145/2487575.2488200) | Google | FTRL 在线学习 |
| 2010 | [Web-Scale Bayesian Click-Through Rate Prediction](https://dl.acm.org/doi/10.1145/1835804.1835848) | Microsoft | 贝叶斯 CTR 预估 |

---

## Embedding 与表示学习

| 年份 | 论文 | 机构 | 核心贡献 |
|------|------|------|---------|
| 2018 | [Billion-scale Commodity Embedding for E-commerce Recommendation](https://arxiv.org/abs/1803.02349) | Alibaba | 大规模 Embedding |
| 2019 | [Real-time Attention Based Look-alike Model](https://arxiv.org/abs/1906.05022) | Tencent | 实时 Lookalike |
| 2016 | [Deep Neural Networks for YouTube Recommendations](https://dl.acm.org/doi/10.1145/2959100.2959190) | Google | YouTube 推荐系统 |

---

## 阅读建议

```
入门路线:
  1. LR → GBDT+LR (Facebook 2014)
  2. FM → DeepFM
  3. Wide & Deep (Google 2016)
  4. DIN → DIEN (Alibaba 2018-2019)
  5. ESMM (多任务学习)
  6. MMOE → PLE

进阶路线:
  7. 出价优化 (RL Bidding)
  8. 在线学习 (FTRL)
  9. 联邦学习
  10. 竞价机制理论
```
