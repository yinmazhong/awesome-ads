# 面试 Q&A Part 1: 行业基础 + 生态

## 一、行业基础 (Industry Basics)

### 1.1 广告发展史

#### Q1: 请简述在线广告的发展历程和各阶段的核心特征 / Describe the evolution of online advertising.

1. **横幅广告 (Banner, 1994–2000)**：AT&T 在 HotWired 投放第一个 Banner，CPM 固定价格，人工交易。The first banner ad was placed by AT&T on HotWired in 1994, sold at fixed CPM rates.
2. **搜索广告 (Search, 2000–2008)**：Google AdWords 开创关键词竞价，CPC 计费，质量分机制。Google AdWords pioneered keyword auction with CPC pricing and Quality Score.
3. **广告网络 (Ad Network, 2003–2009)**：聚合长尾流量，按受众售卖，透明度低。Aggregated long-tail inventory, sold by audience, low transparency.
4. **程序化购买 (RTB, 2009–2015)**：实时竞价，DSP/SSP/Exchange 生态，数据驱动。Real-time bidding, DSP/SSP/Exchange ecosystem, data-driven.
5. **移动广告 (Mobile, 2012+)**：信息流广告成主流，原生广告兴起。In-feed and native ads became mainstream.
6. **智能化+隐私 (AI & Privacy, 2018+)**：深度学习驱动 oCPM；GDPR/ATT 重塑行业。Deep learning powers oCPM; privacy regulations reshape the industry.

---

#### Q2: RTB 的完整流程？各角色职责？ / Complete RTB process and roles?

**流程 (~100ms)**：用户访问 → SSP 发请求到 Ad Exchange → Exchange 广播 Bid Request (OpenRTB) 给多个 DSP → DSP ~50ms 内完成用户识别/人群匹配/CTR-CVR 预估/出价 → Exchange 竞价选胜出者 → 广告展示 → 效果追踪。

**Roles**: Advertiser (budget/creative/goals), DSP (bidding/targeting/budget mgmt), Ad Exchange (auction), SSP (inventory mgmt/yield optimization), DMP (data/audience segments).

---

#### Q3: 程序化购买 vs 传统购买的优势？ / Advantages of programmatic over traditional buying?

| 维度 | 传统 Traditional | 程序化 Programmatic |
|------|-----------------|-------------------|
| 交易 | 人工谈判 IO 合同 | 自动化竞价 |
| 效率 | 周级 | 毫秒级 |
| 定向 | 按版位 | 按人群/行为 |
| 优化 | 手动调整 | 算法实时优化 |

Core: **Data-driven + Real-time + Precise targeting + Automated optimization**.

---

#### Q4: iOS ATT 的影响？行业如何应对？ / Impact of iOS ATT and industry response?

**影响 Impact**: IDFA 授权率仅 ~20-30%；Retargeting 效果大降；确定性归因失效转向 SKAN（24-48h 延迟，仅 64 转化值）；Meta 估计广告主损失 ~$100 亿/年。

**应对 Response**: SKAdNetwork、概率归因、第一方数据 (CDP)、Conversions API 服务端回传、预测模型补全转化、上下文定向复兴。

---

#### Q5: 信息流广告 vs 搜索广告？ / In-feed ads vs search ads?

| 维度 | 搜索广告 Search | 信息流 In-Feed |
|------|---------------|---------------|
| 意图 | 主动搜索，明确 | 被动浏览，模糊 |
| 触发 | 关键词 | 算法推荐 |
| 形式 | 文字链 | 图文/视频/原生 |
| 技术 | Query 理解/匹配/质量分 | CTR/CVR 预估/推荐 |
| 计费 | CPC/oCPC | oCPM |

Search captures **existing demand**; in-feed **creates demand**. Search has higher CVR but lower volume.

---

### 1.2 核心指标

#### Q6: CTR 和 CVR？ / What are CTR and CVR?

- **CTR** = Clicks / Impressions × 100%（信息流 1-3%, 搜索 3-10%, Banner 0.1-0.5%）
- **CVR** = Conversions / Clicks × 100%（电商 1-5%, App 下载 10-30%）

CTR reflects creative quality + targeting; CVR reflects landing page + product-market fit.

---

#### Q7: eCPM？不同计费模式下如何计算？ / eCPM under different billing models?

eCPM = 每千次展示有效收入，**广告排序核心指标**。

| 模式 | 公式 |
|------|------|
| CPM | eCPM = CPM 出价 |
| CPC | eCPM = CPC × pCTR × 1000 |
| CPA | eCPM = CPA × pCTR × pCVR × 1000 |
| oCPM | eCPM = bid × pCTR × pCVR × 1000 |

Unifies different billing models for comparable ranking. Higher eCPM = more revenue per impression.

---

#### Q8: ROI vs ROAS？ / Difference between ROI and ROAS?

- **ROI** = (收入 - 成本) / 成本 × 100%（含所有成本）
- **ROAS** = 广告收入 / 广告花费（仅广告成本）

例：花 ¥10,000 赚 ¥30,000 → ROAS=300%, ROI=200%。广告行业标准用 **ROAS**。

---

#### Q9: 广告主/媒体/平台各关注什么指标？ / Metrics focus by role?

| 角色 | 核心指标 | 关注点 |
|------|---------|--------|
| 广告主 Advertiser | CPA, ROAS, LTV | 转化效率 |
| 媒体 Publisher | eCPM, 填充率, Ad Load | 收入+体验 |
| 平台 Platform | 总收入, 广告主留存 | 生态健康 |

---

#### Q10: 如何保证报表准确性？ / How to ensure report accuracy?

1. 客户端 vs 服务端**交叉验证**
2. **反作弊过滤** GIVT/SIVT
3. **三方对账**（广告主 vs 媒体 vs 第三方），差异 ±5%
4. 统一**归因窗口**
5. 数据管道**监控告警**
6. **MRC 认证**第三方审计（IAS, DoubleVerify, 秒针）

---

### 1.3 商业模式

#### Q11: CPM/CPC/CPA/oCPM 区别？ / Differences between billing models?

| 模式 | 计费 | 风险 | 场景 |
|------|------|------|------|
| CPM | 千次展示 | 广告主 | 品牌广告 |
| CPC | 点击 | 双方 | 搜索广告 |
| CPA | 转化 | 媒体 | App 下载 |
| **oCPM** | 展示计费+转化优化 | 平台优化 | **效果广告主流** |

oCPM: 广告主设目标 CPA → 平台预估 pCTR×pCVR → `bid = CPA × pCVR × 1000`。

---

#### Q12: oCPM 出价逻辑？如何保证成本？ / oCPM bidding logic and cost guarantee?

**出价**: `bid = target_CPA × pCTR × pCVR × 1000`。高转化概率用户出高价。

**成本保障**: PID 控制器实时调整出价系数 + 赔付机制（超标 120% 赔付）+ 模型校准 + Pacing 均匀消耗 + 冷启动保护。

---

#### Q13: 一价 vs 二价竞价？为什么转一价？ / First-price vs second-price? Why the shift?

| | 二价 Second-Price | 一价 First-Price |
|--|------------------|-----------------|
| 扣费 | 第二高+¥0.01 | 实际出价 |
| 策略 | 真实出价最优 | 需 Bid Shading |

转一价原因：Header Bidding 使二价在多层竞价中失效；一价更透明简单。**Bid Shading**: ML 预测"刚好能赢"的最低出价。

---

#### Q14: 品牌广告 vs 效果广告？ / Brand vs performance advertising?

| | 品牌 Brand | 效果 Performance |
|--|-----------|-----------------|
| 目标 | 曝光/认知 | 转化/ROI |
| 计费 | CPM/CPT | oCPM/CPA |
| 占比 | 下降 | >70%，上升 |

Brand = awareness (top funnel); Performance = action (bottom funnel).

---

## 二、广告生态 (Ecosystem)

### 2.1 需求方

#### Q15: DSP 核心功能？竞价流程？ / DSP functions and bidding flow?

功能：人群定向、实时竞价、预算管理、创意管理、效果追踪。

竞价 (~50ms)：Bid Request → 用户识别 → DMP 查询 → 广告召回 → CTR/CVR 预估 → 出价 → 预算检查 → Bid Response。

---

#### Q16: DMP 第一/二/三方数据？ / 1st/2nd/3rd party data?

| 类型 | 来源 | 价值 |
|------|------|------|
| 第一方 1P | 自有 CRM/App/交易 | 最高质量，合规 |
| 第二方 2P | 合作伙伴/清洁室 | 补充盲区 |
| 第三方 3P | 外部供应商 | 规模大，质量参差 |

趋势：隐私趋严，**第一方数据战略**成核心。Trend: 1P data strategy is critical.

---

#### Q17: 常见归因模型？ / Common attribution models?

| 模型 | 逻辑 | 优缺点 |
|------|------|--------|
| 最后点击 Last Click | 100% 归最后渠道 | 简单但忽略上游 |
| 首次点击 First Click | 100% 归首次渠道 | 重视拉新但忽略路径 |
| 线性 Linear | 平均分配 | 公平但无差异 |
| 时间衰减 Time Decay | 越近权重越高 | 较合理但仍是规则 |
| 数据驱动 Data-Driven | Shapley Value | 最准确但需大量数据 |

趋势：规则→数据驱动。GA4 默认数据驱动归因。

---

#### Q18: 国内 vs 海外广告生态？ / China vs overseas ad ecosystem?

| | 海外 Overseas | 国内 China |
|--|-------------|-----------|
| 结构 | 开放生态，独立 DSP/SSP | 围墙花园，平台闭环 |
| 数据 | 跨平台流通 | 数据不出域 |
| 特色技术 | Header Bidding, SKAN | **RTA** |

围墙花园 = 广告主需多平台分别投放，跨平台归因困难。

---

### 2.2 供给方

#### Q19: SSP 核心功能？vs Ad Network？ / SSP functions vs Ad Network?

SSP：广告位管理、收益优化、底价管理、需求源管理。

| | Ad Network | SSP |
|--|-----------|-----|
| 角色 | 中间商低买高卖 | 代表媒体利益 |
| 定价 | 固定/包断 | 实时竞价 |
| 透明度 | 低 | 高 |

---

#### Q20: Waterfall vs Header Bidding？

- **Waterfall**：按历史 eCPM 排序依次请求。✅ 简单 ❌ 非实时最优
- **Header Bidding**：同时请求所有需求源实时竞价。✅ 收入 +20-40% ❌ 延迟增加

---

#### Q21: 媒体如何平衡收入和体验？ / How to balance revenue and UX?

Ad Load 控制（A/B 测试最优密度）+ 频次控制 + 原生广告 + 创意质量把控 + 用户反馈机制 + 长期指标监控（留存/时长）。

---

### 2.3 广告交易

#### Q22: OpenRTB 核心字段？ / Core OpenRTB fields?

**Request**: imp（广告位）, user, device, site/app, regs（法规标志）
**Response**: bid（出价）, adm（素材）, nurl（胜出通知）, crid（创意 ID）

---

#### Q23: PMP vs RTB？

| | RTB Open Auction | PMP |
|--|-----------------|-----|
| 参与者 | 所有 DSP | 受邀 DSP |
| 流量 | 含长尾 | 优质为主 |
| Deal ID | 无 | 有 |

---

#### Q24: 什么是 RTA？ / What is RTA?

**RTA (Real-Time API)**：媒体向广告主 RTA 服务发实时请求 (<50ms)，广告主返回是否参竞/出价调整/人群标签。弥补围墙花园下数据不互通。要求：50ms 响应、万级 QPS、99.99% 可用性。

RTA lets advertisers inject their own data capabilities into the media's ad decision process in real-time, compensating for data isolation in walled gardens.
