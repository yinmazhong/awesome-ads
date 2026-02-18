# 面试 Q&A Part 2: 广告系统 + 数据算法

## 三、广告系统 (Ad System)

### 3.1 投放引擎 (Serving)

#### Q25: 广告投放引擎的核心流程？ / Core process of ad serving engine?

漏斗架构 (~50ms 服务端) / Funnel architecture:

1. **请求解析** (~2ms) — Parse request
2. **召回 Retrieval** (~10ms) — 百万→千级（定向/向量/行为召回）
3. **粗排 Pre-ranking** (~5ms) — 千→百（轻量模型）
4. **精排 Ranking** (~15ms) — 百→十（深度模型，eCPM = bid × pCTR × pCVR × 1000）
5. **机制 Mechanism** (~5ms) — 竞价/预算/频次/Pacing
6. **重排 Re-ranking** (~3ms) — 多样性/生态调控

Total: ~48ms server + ~25ms network + ~25ms client rendering ≈ 100ms.

---

#### Q26: 召回阶段常见策略？ / Common retrieval strategies?

1. **定向召回 Targeting-based**：倒排索引匹配定向条件 — Inverted index matching
2. **向量召回 Vector-based**：ANN 检索 (Faiss/HNSW) — Approximate nearest neighbor
3. **行为召回 Behavior-based**：基于用户历史行为 — Based on user history
4. **热门召回 Popularity-based**：高 CTR 广告，用于冷启动 — For cold start
5. **实时召回 Real-time**：当前 Session 行为触发 — Session-based triggers

关键：召回率（好广告不漏）+ 延迟效率。Key: recall rate + latency budget.

---

#### Q27: eCPM 排序公式？各因子含义？ / eCPM ranking formula?

**基础**: `eCPM = bid × pCTR × pCVR × 1000`

| 因子 Factor | 含义 Meaning | 来源 Source |
|------------|-------------|------------|
| bid | 广告主出价 | 目标 CPA/CPC |
| pCTR | 预估点击率 | CTR 模型 |
| pCVR | 预估转化率 | CVR 模型 |

**扩展**: `score = eCPM × quality_score × ecosystem_factor`

quality_score 保证广告质量；ecosystem_factor 实现平台调控（新广告主扶持、行业均衡）。

---

#### Q28: 预算控制 (Pacing) 如何实现？ / How is budget pacing implemented?

目标：预算均匀分配到投放时段，避免过早耗尽。

**PID 控制器**（最常用）：监控实际消耗 vs 目标消耗偏差，动态调整出价系数 λ。
- P（比例）：当前偏差 → `α × e(t)`
- I（积分）：累积偏差 → 消除稳态误差
- D（微分）：偏差变化率 → 防止超调
- 消耗过快 → 降 λ → 降出价；过慢 → 升 λ → 升出价

PID controller monitors spend rate deviation and dynamically adjusts bid multiplier λ.

---

#### Q29: 新广告冷启动如何解决？ / How to solve cold-start for new ads?

1. **Explore & Exploit**：Thompson Sampling 分配探索流量
2. **先验估计 Prior**：同类广告（同行业/广告主）历史数据作初始值
3. **特征迁移 Transfer**：Embedding 迁移学习
4. **保守出价**：冷启动期保守策略，逐步放量
5. **加速学习**：前期提高曝光权重快速积累数据
6. **创意预估**：基于图片/文案特征预估初始 CTR

通常需 20-50 个转化才能进入稳定期。Typically needs 20-50 conversions to stabilize.

---

#### Q30: 广告系统如何 100ms 内响应？ / How to achieve sub-100ms response?

1. **漏斗架构**：逐层过滤减少计算量
2. **并行处理**：特征获取/多路召回并行
3. **多级缓存**：本地 (Caffeine) → Redis → DB，命中率 >95%
4. **模型优化**：蒸馏/量化 (INT8)/TensorRT
5. **预计算**：离线预算用户/广告特征
6. **异步化**：非关键路径异步（日志/监控）
7. **就近部署**：多机房，用户就近接入

---

### 3.2 定向 (Targeting)

#### Q31: 广告定向维度？实现原理？ / Targeting dimensions and implementation?

| 维度 Dimension | 实现 Implementation | 数据 Data |
|---------------|--------------------|---------| 
| 地域 Geo | IP 地理库/GPS | 设备信息 |
| 人口属性 Demo | 注册信息+模型预测 | 第一方数据 |
| 兴趣 Interest | 行为数据挖掘标签 | 浏览/搜索/购买 |
| 行为 Behavior | 历史行为序列 | 点击/转化日志 |
| 重定向 Retargeting | 像素/SDK 追踪 | 广告主埋点 |
| Lookalike | 种子用户扩展 | 种子包+模型 |
| 上下文 Context | NLP 分析页面内容 | 页面内容 |

技术：**倒排索引**快速匹配 + **布尔表达式 (DNF)** 精确匹配。

---

#### Q32: Lookalike 扩量原理？ / Principle of Lookalike expansion?

1. 广告主提供**种子用户** (seed) — 如已付费用户
2. 分析种子**共同特征** — 行为/兴趣/人口属性
3. 全量用户中找**相似人群** — Embedding 余弦相似度 / 分类模型
4. 广告主选**扩展比例** (1%/5%/10%) — 越小越精准，越大量越多

Lookalike: Analyze seed users' common features → Find similar users via embedding similarity or classification models → Control precision with expansion ratio.

---

#### Q33: 重定向 (Retargeting) 如何实现？ / How is retargeting implemented?

部署追踪像素/SDK → 记录浏览/加购行为 → 构建人群（如"加购未购买"）→ 定向投放 → **动态创意**（展示用户浏览过的具体商品）。

隐私时代挑战：Cookie/IDFA 追踪受限，需转向第一方数据 + 服务端追踪。

Deploy pixel/SDK → Record behavior → Build segments (e.g., "added to cart but didn't buy") → Target with dynamic creatives showing viewed products.

---

#### Q34: 定向系统技术实现？ / Technical implementation of targeting?

**倒排索引 Inverted Index**：每个标签维护广告 ID 列表，请求到来取交集。O(1) 查找。适合标签型定向。

**布尔表达式 Boolean Expression**：`(age ∈ [25,35]) AND (city ∈ {北京}) AND (interest ∈ {美妆})`。DNF 索引加速。适合复杂组合。

实际：两者结合 — 倒排粗筛 + 布尔精确匹配。Production: combine both for coarse + precise filtering.

---

#### Q35: 定向太窄/太宽的问题？如何平衡？ / Too narrow/broad targeting?

- **太窄**：曝光不足，消耗不出去 — Not enough reach
- **太宽**：CPA 高，无效曝光多 — Wasted impressions

平衡：智能放量（系统自动扩展）+ Lookalike + oCPM 自动优化 + 分层测试（核心/扩展/探索人群分配不同预算）+ 排除已转化用户。

---

### 3.3 创意 (Creative)

#### Q36: 广告创意审核流程和技术？ / Ad creative review process and tech?

流程：素材提交 → **机器预审** (秒级, AI+规则, 处理 >90%) → 通过/待人审 → **人工审核** (分钟~小时) → 上线/拒绝反馈。

技术：OCR 文字识别、图像分类（色情/暴力）、NLP 敏感词、商标识别、视频关键帧抽取。机审持续学习人审结果。

---

#### Q37: 动态创意优化 (DCO) 原理？ / Principle of DCO?

将创意拆解为**组件**（标题/图片/描述/CTA）→ 自动**组合**生成多套创意 → **Multi-Armed Bandit** 算法实时测试 → 流量分配给表现最好的组合。实现千人千面。

DCO decomposes creatives into components, auto-combines them, and uses MAB algorithms to allocate traffic to the best-performing combinations.

---

#### Q38: 创意疲劳是什么？如何应对？ / What is creative fatigue? How to address?

用户反复看同一创意后 CTR 逐渐下降。Users see the same creative repeatedly, CTR declines.

应对：频次控制 + 定期更新素材 + AIGC 批量生成变体 + 动态创意自动轮换 + 监控 CTR 衰减曲线及时预警。

---

#### Q39: AIGC 在广告创意中的应用？ / AIGC applications in ad creatives?

- **文案生成**：LLM 生成标题/描述/脚本（已成熟）
- **图片生成**：Stable Diffusion/DALL-E 生成广告图（快速成熟）
- **视频生成**：Sora/Runway 生成视频（早期）
- **数字人**：AI 数字人代替真人（已商用）
- **个性化**：根据用户特征生成千人千面创意

效率提升 10-100x，成本大幅降低。Efficiency up 10-100x, cost significantly reduced.

---

### 3.4 计费与反作弊 (Billing & Anti-Fraud)

#### Q40: 常见广告作弊手段？如何检测？ / Common ad fraud and detection?

| 作弊 Fraud | 说明 | 检测 Detection |
|-----------|------|---------------|
| 机器刷量 Bot | 爬虫/脚本模拟 | IP/UA 异常、行为模式 |
| 设备农场 Device Farm | 大量真机刷量 | 设备聚集度、GPS 异常 |
| 点击注入 Click Injection | 劫持归因 | 点击-安装时间异常短 |
| SDK 欺骗 SDK Spoofing | 伪造信号 | 加密校验、签名验证 |
| 归因劫持 Attribution Fraud | 抢夺归因 | 时间戳分析 |

分层：规则引擎 (GIVT) → ML 模型 (SIVT) → 人工审核。Layered: rules → ML → human review.

---

#### Q41: 广告计费系统架构？ / Ad billing system architecture?

请求日志 → Kafka → **Flink 实时计费**（去重/反作弊过滤/预算扣减）→ 计费数据库 → 报表聚合。

关键：**幂等性**（防重复扣费）、**最终一致性**、**实时预算控制**。

Key: idempotency (prevent double-charging), eventual consistency, real-time budget control.

---

#### Q42: GIVT vs SIVT？ / What are GIVT and SIVT?

- **GIVT (General Invalid Traffic)**：通用无效流量，可通过规则识别。如爬虫、已知数据中心 IP、预加载。
- **SIVT (Sophisticated Invalid Traffic)**：复杂无效流量，需 ML 检测。如设备农场、点击注入、归因劫持。

GIVT is identifiable by rules (bots, known datacenter IPs); SIVT requires ML detection (device farms, click injection).

---

#### Q43: 广告归因流程？ / Ad attribution process?

广告曝光/点击 → 记录归因信息（设备 ID/时间戳/广告 ID）→ 用户转化 → **匹配归因**（确定性：ID 精确匹配；概率性：IP/设备指纹模糊匹配）→ 归因到对应广告计划。

归因窗口：通常 7 天点击 + 1 天曝光。Attribution window: typically 7-day click + 1-day view.

---

#### Q44: iOS ATT 对归因的影响？SKAN 限制？ / ATT impact on attribution? SKAN limitations?

SKAN 限制 / Limitations:
- 转化数据延迟 **24-48h** — Delayed conversion data
- 仅 **64 个转化值** (6 bit) — Only 64 conversion values
- **无用户级数据**，只有聚合 — No user-level data, aggregated only
- 无法做**实时优化** — Cannot optimize in real-time
- **单一归因来源** — Single attribution source (last ad network)

---

#### Q45: 如何做广告数据对账？常见差异原因？ / Ad data reconciliation? Common discrepancy causes?

对账：广告主报表 vs 媒体报表 vs 第三方监测，差异率控制 ±5%。

常见差异原因：
- **统计口径不同**：曝光定义（服务端 vs 客户端可见）
- **归因窗口不同**：7 天 vs 30 天
- **时区差异**：UTC vs 本地时区
- **反作弊过滤差异**：各方过滤规则不同
- **数据延迟**：实时 vs T+1

---

## 四、数据与算法 (Data & Algorithm)

### 4.1 用户画像 (User Profile)

#### Q46: 用户画像系统整体架构？ / User profile system architecture?

数据采集（行为日志/第三方）→ 数据处理（Spark 离线 + Flink 实时）→ 标签计算（规则+模型）→ 存储（HBase/Redis）→ 服务（在线查询 API, P99 <5ms）。

Data collection → Processing (Spark offline + Flink realtime) → Tag computation (rules + models) → Storage (HBase/Redis) → Serving (API, P99 <5ms).

---

#### Q47: 常见用户标识？ID Mapping 如何实现？ / User identifiers and ID Mapping?

标识：IDFA/GAID/OAID、Cookie、手机号、邮箱、自有 User ID。

**ID Mapping**：构建 **ID Graph**，确定性匹配（登录关联）+ 概率性匹配（设备指纹），图算法（连通分量）合并多标识到同一用户。

Build an **ID Graph** using deterministic matching (login association) + probabilistic matching (device fingerprint), merge via graph algorithms (connected components).

---

#### Q48: 离线标签 vs 实时标签？ / Offline vs real-time tags?

| | 离线标签 Offline | 实时标签 Real-time |
|--|-----------------|-------------------|
| 计算 | Spark 批处理 | Flink 流处理 |
| 更新 | T+1 | 秒~分钟级 |
| 示例 | 长期兴趣、消费能力 | 当前意图、实时行为 |
| 存储 | HBase/Hive | Redis |

---

#### Q49: 人群包技术实现？Bitmap vs Bloom Filter？ / Audience segment implementation?

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| **RoaringBitmap** | 精确、支持集合运算、压缩率高 | 需整数 ID | **主流方案** |
| **Bloom Filter** | 空间极小 | 有误判、不支持删除 | 大规模粗筛 |

RoaringBitmap 支持 AND/OR/NOT，用于定向匹配。Production standard.

---

### 4.2 CTR/CVR 预估 (Prediction)

#### Q50: CTR 预估模型演进？核心思想？ / CTR model evolution?

`LR → FM → FFM → Wide&Deep → DeepFM → DIN → DIEN → MMOE/PLE`

| 模型 Model | 核心 Core Idea |
|-----------|---------------|
| LR | 线性模型，手工特征交叉 Linear, manual feature crossing |
| FM | 自动二阶交叉，隐向量内积 Auto 2nd-order interaction via latent vectors |
| Wide&Deep | Wide(记忆) + Deep(泛化) Wide(memorization) + Deep(generalization) |
| DeepFM | FM 替代 Wide，自动交叉 FM replaces Wide for auto crossing |
| DIN | **Attention** 建模用户兴趣与候选广告相关性 Attention on user behavior |
| DIEN | 序列模型捕捉兴趣**演化** Sequential model for interest evolution |
| MMOE/PLE | **多任务学习**，多目标联合优化 Multi-task learning |

---

#### Q51: FM 原理？复杂度优化？ / FM principle and complexity optimization?

FM 为每个特征学习 k 维隐向量 v，二阶交叉 `<vi, vj>` 代替 wij。

原始 O(kn²) → 数学变换优化为 **O(kn)**：`Σ<vi,vj>xixj = 1/2[(Σvixi)² - Σ(vixi)²]`

FM learns k-dim latent vectors per feature. Complexity reduced from O(kn²) to O(kn) via algebraic reformulation.

---

#### Q52: DIN 核心创新？ / Core innovation of DIN?

传统模型对用户历史行为做 sum/mean pooling，丢失与候选广告的相关性。

DIN 引入 **Attention 机制**：根据候选广告对历史行为**加权** — 相关行为权重高，不相关权重低。实现"千人千面"兴趣表达。

DIN introduces **attention** over user behavior history, weighting each historical item by its relevance to the candidate ad. This enables personalized interest representation.

---

#### Q53: CVR 样本选择偏差？ESMM 如何解决？ / CVR sample selection bias? ESMM solution?

**问题 Problem**：CVR 模型只能在**点击样本**上训练（只有点击后才知道是否转化），但推理时需对**所有曝光**预估。训练/推理样本空间不一致。

**ESMM 方案**：在全量曝光样本上联合训练 pCTR 和 pCVR，利用 `pCTCVR = pCTR × pCVR` 关系，用 CTCVR 作为监督信号，消除偏差。

ESMM jointly trains pCTR and pCVR on all impression samples, using `pCTCVR = pCTR × pCVR` as supervision to eliminate selection bias.

---

#### Q54: 什么是模型校准？为什么广告需要？ / What is model calibration? Why needed?

校准确保预估概率的**绝对值准确**（不仅排序准确）。

广告需要因为出价公式 `bid = CPA × pCVR × 1000` **直接使用预估值计算金额**。pCVR 偏高 → 出价偏高 → 成本超标。

方法：Isotonic Regression、Platt Scaling。指标：ECE (Expected Calibration Error)。

Calibration ensures predicted probabilities are accurate in absolute terms, not just ranking. Critical because bid formulas directly use predicted values to calculate monetary amounts.

---

#### Q55: 在线学习 vs 离线学习？FTRL？ / Online vs offline learning? FTRL?

| | 离线 Offline | 在线 Online |
|--|-------------|------------|
| 更新 | 天/小时级 | 实时/分钟级 |
| 数据 | 全量历史 | 流式增量 |
| 优势 | 稳定 | 捕捉实时变化 |

**FTRL (Follow The Regularized Leader)**：在线学习算法，L1 正则产生**稀疏解**（特征选择），适合高维稀疏广告特征。

---

#### Q56: 多任务学习 MMOE/PLE 在广告中的应用？ / Multi-task learning in ads?

广告需同时预估多目标（CTR/CVR/停留时长/付费）。

- **MMOE**：多个 Expert + Gate 为每个任务动态分配 Expert 权重
- **PLE**：区分 task-specific 和 shared Expert，减少任务冲突

优势：共享底层表征减少计算；任务间信息互补提升效果。

MMOE uses multiple experts with task-specific gates; PLE separates task-specific and shared experts to reduce task conflicts.

---

### 4.3 出价 (Bidding)

#### Q57: PID 控制在预算 Pacing 中的应用？ / PID control in budget pacing?

（详见 Q28）PID 输出出价调整系数 λ = f(P, I, D)。`实际出价 = 原始出价 × λ`。P 响应当前偏差，I 消除累积误差，D 防止超调。

---

#### Q58: 智能出价的成本控制机制？ / Cost control in smart bidding?

1. PID Pacing 控制消耗速率
2. 赔付机制（超标 120% 赔付）
3. 模型校准（预估准确性）
4. 出价截断（最高出价上限）
5. 冷启动保护
6. 多目标约束（成本+量级联合优化）

---

#### Q59: 强化学习在出价中的应用？ / RL in bidding?

将出价建模为**序列决策问题**：状态（预算余额/时段/竞争环境）→ 动作（出价系数）→ 奖励（转化/成本）。

方法：DQN、DDPG、Constrained RL（带约束的强化学习，满足成本约束）。

优势：全局最优（考虑未来预算分配），而非贪心的逐次最优。

RL models bidding as a sequential decision problem: state (budget/time/competition) → action (bid multiplier) → reward (conversions/cost). Achieves globally optimal allocation vs greedy per-impression optimization.

---

#### Q60: 预算分配策略？ / Budget allocation strategies?

1. **计划间分配**：根据各计划 ROI 动态调整预算（高 ROI 追加，低 ROI 缩减）
2. **渠道间分配**：跨平台（抖音/微信/百度）按效果分配
3. **时段分配**：根据流量和转化率分时段（晚高峰多分配）
4. **预留池**：保留 10-20% 预算用于探索和应急

---

### 4.4 推荐广告融合 (Rec-Ads Blend)

#### Q61: 推荐系统 vs 广告系统核心区别？ / Rec system vs ad system?

| | 推荐 Rec | 广告 Ads |
|--|---------|---------|
| 目标 | 用户体验（点击/时长） | 收入 (eCPM) |
| 内容 | 平台内容 | 广告主付费内容 |
| 排序 | 相关性/兴趣 | eCPM（含出价） |
| 约束 | 多样性/新鲜度 | 预算/频次/成本 |

---

#### Q62: 常见混排策略？ / Common blending strategies?

| 策略 | 方法 | 优缺点 |
|------|------|--------|
| 固定位置 Fixed | 第 N 条为广告 | ✅ 简单 ❌ 不灵活 |
| 概率插入 Probabilistic | 按概率插入 | ✅ 灵活 ❌ 体验不稳定 |
| 联合排序 Unified | 广告+内容统一排序 | ✅ 最优 ❌ 复杂 |

---

#### Q63: Ad Load 如何确定最优值？ / How to determine optimal Ad Load?

A/B 测试不同 Ad Load (5%/10%/15%/20%)，观察**短期收入**和**长期留存**。存在拐点：Ad Load 超过某值后收入增长放缓但留存加速下降。最优点在**收入边际收益 = 留存边际损失**处。

---

### 4.5 搜索广告 (Search Ads)

#### Q64: 搜索广告核心流程？ / Core search ads process?

Query 输入 → **Query 理解**（分词/意图/改写）→ **关键词匹配**（广泛/短语/精确）→ 广告召回 → **质量分**计算 → 排序 (`Ad Rank = bid × QS`) → 扣费 → 展示。

---

#### Q65: 关键词匹配类型？ / Keyword match types?

| 类型 Type | 说明 | 流量 Volume | 精准度 Precision |
|----------|------|-----------|-----------------|
| 精确 Exact | Query 完全一致 | 小 | 最高 |
| 短语 Phrase | 包含关键词 | 中 | 中 |
| 广泛 Broad | 语义相关即可 | 大 | 低 |

---

#### Q66: 质量分？如何影响排名和扣费？ / Quality Score impact on ranking and pricing?

质量分 QS = f(历史 CTR, 创意相关性, 落地页体验)。

- 排名：`Ad Rank = bid × QS`。QS 高可用更低出价获更好位置。
- 扣费 (GSP)：`CPC = 下一名 Ad Rank / 自己 QS + ¥0.01`

QS incentivizes relevant, high-quality ads. Higher QS = lower cost for same position.

---

#### Q67: GSP 竞价机制？ / How does GSP auction work?

**GSP (Generalized Second-Price)**：每个广告主支付刚好超过下一名所需的最低价格。

排名按 Ad Rank = bid × QS。扣费：`CPC_i = Ad Rank_{i+1} / QS_i + ¥0.01`。

与 VCG 的区别：GSP 不是激励相容的（真实出价不一定最优），但实现简单，是搜索广告的标准机制。

---

#### Q68: Query 理解包含哪些技术？ / Query understanding techniques?

1. **分词 Tokenization**：中文分词（jieba/自研）
2. **意图识别 Intent**：导航/信息/交易意图分类
3. **Query 改写 Rewriting**：同义词替换、纠错、扩展
4. **实体识别 NER**：品牌、产品、地点等
5. **语义理解**：BERT/大模型深层语义匹配

Modern: LLM-based query understanding for deeper semantic matching beyond keyword level.
