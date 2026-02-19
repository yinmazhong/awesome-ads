# Meta DSP 学习笔记（账户体系 / 风控封禁 / 人群包与冷启动）

本文用于系统性梳理 Meta 投放（常说的“Meta DSP”）的关键知识：

- **投放账户体系**：BM、广告账号、像素、域名、Page、Developer App、System User、Token 的关系
- **风控与封禁**：常见处罚层级、触发原因、排查与申诉入口
- **人群包管理**：Customer List、Website Custom Audience、Lookalike、价值优化，核心目标是**缩短冷启动**

---

## 一、账户与资产体系（一定要先搞清楚）

### 1.1 关键对象（概念表）

| 对象 | 作用 | 常见风险点 |
|------|------|------------|
| **Business Manager (BM)** | 资产容器与权限中心（广告账号、像素、域名、Page、系统用户等） | BM 被限制会波及多种资产 |
| **Ad Account（广告账号）** | 实际投放与计费载体 | 最常见被封层级 |
| **Page / Instagram Account** | 广告展示主体（常用于品牌/落地一致性） | Page 质量、内容合规 |
| **Pixel** | Web 事件采集与归因/优化信号（PWA 场景核心） | 像素/域名资产被限制会影响事件回传 |
| **Domain（域名）** | Web 落地页与验证 | 域名违规/强跳转/cloaking 风险很高 |
| **Developer App（App ID）** | API 权限载体（Marketing API、部分自动化能力） | 权限审查、token 吊销、主体级风控连带 |
| **System User** | 在 BM 内代表“系统身份”做 API 调用 | 需要正确授权到广告账号/像素等资产 |
| **Access Token** | API 调用凭证（短期/长期/系统用户 token） | 过期、权限不足、被吊销 |

### 1.2 一张“关系图”（用来排查问题）

```
BM
 ├── Ad Account(s)  ← 投放/计费/封禁常发生在这里
 ├── Pixel(s)       ← PWA/网站事件、优化信号
 ├── Domain(s)      ← 域名验证、落地合规
 ├── Page / IG      ← 展示主体
 ├── System User(s) ← API 调用身份
 └── Developer App  ← 权限申请/授权（App Review/Advanced Access 相关）
```

---

## 二、风控/封禁：分层理解（决定“会不会连坐”）

### 2.1 封禁/限制的常见层级

| 层级 | 常见表现 | 典型影响范围 |
|------|----------|--------------|
| **Ad（单条广告）** | Rejected/Disapproved | 只影响该素材/落地/广告 |
| **Ad Set / Campaign** | 限制学习、无法投放 | 影响一组投放 |
| **Ad Account** | Disabled/Restricted | 整个账号无法投放 |
| **BM（Business）** | Restricted/Disabled | 多账号、多资产一起受限 |
| **Developer/App/主体** | 权限撤销、token 吊销、API 不可用 | 影响自动化、人群包上传、CAPI 等链路 |

### 2.2 “广告被封会不会波及 Developer App（App ID）？”

这里的 **App** 指的是 **Meta Developer 账号里的 App（App ID）**，用于 Marketing API / 自定义受众 / 自动化等。

- **不一定**：如果只是单条广告违规或轻度拒登，通常不会动到 App ID。
- **可能波及**：如果升级到 **广告账号/BM 层**的严重处罚（博彩、欺诈、规避审核、cloaking 等），更容易触发主体级风控，进而连带 Developer App、System User、Token、业务验证状态等。

### 2.3 “能否绕过 App 审核去上传自定义人群？”

结论是：**不能绕过合规要求，只能换路径，让链路不依赖 Developer App 的权限审查。**

- **用 Marketing API 上传 Customer List**：通常离不开 Developer App 权限、token、BM 验证等。
- **用 Ads Manager 界面手工上传**：往往可以不走 Developer App 审核链路，但仍可能要求 BM/账户层面的验证与条款合规。

---

## 三、人群包与冷启动：核心目标是“让新账号快速收敛”

### 3.1 冷启动为什么慢？

新广告账号/新像素/新事件目标上线后，模型需要在短时间内看到足够多的“高质量转化样本”，才能稳定优化。你能做的就是：

- 让 **受众** 更像“高价值人群”（种子更好）
- 让 **转化回传** 更稳定、更准确（信号更好）

### 3.2 人群的三大类（按可迁移性排序）

| 人群类型 | 怎么来 | 是否适合“封号后迁移到新账号” | 说明 |
|---------|--------|------------------------------|------|
| **Customer List Custom Audience** | 上传 email/phone 等（hash） | ✅ 很适合 | 你的用户资产最可迁移，但合规要求高 |
| **Website Custom Audience** | Pixel/CAPI 事件沉淀 | ✅ 适合 | 依赖像素/域名资产稳定性 |
| **Engagement Custom Audience** | Page/IG 互动 | ✅ 适合 | 品牌向更常见 |

### 3.3 Lookalike（相似受众）是冷启动“加速器”

基本打法：

1. 用高质量种子（例如 `Payers_30D`、`High_LTV`）建 Custom Audience
2. 基于种子建 Lookalike（1% 起步）
3. 冷启动先用 `Lead/CompleteRegistration` 拉量，稳定后切 `Purchase/value`

### 3.4 PWA 场景：Pixel + CAPI 才是优化信号主干

如果你投的是 PWA（Web），尤其是“Meta 账号暂不绑定 App”的限制下：

- 优化信号主干应当是 **Meta Pixel + Conversions API (CAPI)**
- 关键工程点：
  - **去重**：Pixel 与 CAPI 使用相同 `event_id` 做 dedup
  - **匹配率**：尽可能提供 email/phone hash、ip、ua 等，提升匹配
  - **口径稳定**：事件命名/币种/金额字段要长期一致，便于跨账号迁移

---

## 四、封禁后“最快恢复投放”的操作清单（可直接执行）

### 4.1 资产隔离（长期策略）

- 高风险测试与核心业务分离：**BM / Ad Account / Domain / Pixel / Page**
- 自动化链路（Developer App / System User / Token）尽量绑定在“更干净”的主体与资产上

### 4.2 新账号冷启动（48-72h 目标）

1. 准备种子：Customer List 或 Website 事件受众
2. 建 Lookalike：1% 起
3. 事件策略：先 `Lead/CompleteRegistration` 跑通，再切 `Purchase/value`
4. 回传策略：Pixel + CAPI 双写 + `event_id` 去重
5. 稳定策略：前 48h 避免频繁大幅改预算/频繁切目标

---

## 五、Meta 官方文档链接（我找到的都记录在这里）

### 5.1 政策与高风险类目（风控的“法条”入口）

- Advertising Standards（总入口）
  - https://transparency.meta.com/policies/ad-standards/
- Online Gambling and Games（博彩类目：通常需要授权/验证）
  - https://transparency.meta.com/policies/ad-standards/restricted-goods-services/gambling-games/

### 5.2 账户受限/封禁排查与申诉

- Troubleshoot a Disabled or Restricted Account（受限账号排查/申诉入口）
  - https://www.facebook.com/business/help/422289316306981
- Account Quality（账户质量面板入口）
  - https://business.facebook.com/accountquality

### 5.3 业务验证（Business Verification）

- About Business Verification in Meta Business Suite
  - https://www.facebook.com/business/help/1095661473946872
- Verify Your Business in Meta Business Suite
  - https://www.facebook.com/business/help/2058515294227817
- Upload official documents to verify your business
  - https://www.facebook.com/business/help/159334372093366
- When to Use Domain Verification to Verify Your Business
  - https://www.facebook.com/business/help/245311299870862

### 5.4 Pixel（Web 事件采集）

- Meta Pixel（开发者文档入口）
  - https://developers.facebook.com/docs/meta-pixel/
- Conversion Tracking（标准转化事件实现）
  - https://developers.facebook.com/docs/meta-pixel/implementation/conversion-tracking/

### 5.5 Conversions API（服务端回传，抗丢与稳定性关键）

- Conversions API（总入口）
  - https://developers.facebook.com/docs/marketing-api/conversions-api/
- Get Started（入门）
  - https://developers.facebook.com/docs/marketing-api/conversions-api/get-started/
- Customer Information Parameters（匹配率相关参数）
  - https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/customer-information-parameters/
- Dedup（Pixel 与 Server 事件去重）
  - https://developers.facebook.com/docs/marketing-api/conversions-api/deduplicate-pixel-and-server-events/

### 5.6 自定义受众（Custom Audiences）

- Customer File Custom Audiences（列表人群，上传 hash）
  - https://developers.facebook.com/docs/marketing-api/audiences/guides/custom-audiences/
- Custom Audience（对象 reference）
  - https://developers.facebook.com/docs/marketing-api/reference/custom-audience/
- Custom Audience Users（上传用户接口 reference）
  - https://developers.facebook.com/docs/marketing-api/reference/custom-audience/users/
- Website Custom Audiences（网站行为人群）
  - https://developers.facebook.com/docs/marketing-api/audiences/guides/website-custom-audiences/
- Engagement Custom Audiences（互动人群）
  - https://developers.facebook.com/docs/marketing-api/audiences/guides/engagement-custom-audiences/

### 5.7 Custom Audiences（业务侧帮助文档：上传、人群格式、hash）

- About Hashing Customer Information
  - https://www.facebook.com/business/help/112061095610075
- About Custom Audiences
  - https://www.facebook.com/business/help/744354708981227
- Create a Customer List Custom Audience
  - https://www.facebook.com/business/help/170456843145568
- About Customer List Custom Audiences
  - https://www.facebook.com/business/help/341425252616329

### 5.8 Marketing API / 授权 / System User

- Marketing API Reference
  - https://developers.facebook.com/docs/marketing-api/reference/
- Authorization（Marketing API 授权）
  - https://developers.facebook.com/docs/marketing-api/get-started/authorization/
- System User Access Token Handling
  - https://developers.facebook.com/docs/marketing-api/guides/smb/system-user-access-token-handling/
- System Users（对象与权限概念）
  - https://developers.facebook.com/docs/marketing-api/system-users/