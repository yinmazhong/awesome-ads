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

### 3.5 人群包“实战打法”：从 0 到 1 搭一个可迁移的受众矩阵

下面给你一套可以直接照搬的“受众矩阵”，目标是：

- **冷启动**：让新账号在 48-72 小时内进入可优化状态
- **可迁移**：即使换广告账号，仍能复用“人群资产 + 信号工程”

#### 3.5.1 受众矩阵（Audience Matrix）

| 层 | 受众类型 | 例子 | 主要作用 | 是否容易跨账号复用 |
|---|----------|------|----------|------------------|
| L0 | 广泛/兴趣 | Broad、兴趣包 | 拉量、探索 | ⚠️ 受账号学习影响较大 |
| L1 | Website Custom Audience | 7D/14D 访问者、加购者 | 召回、稳定转化 | ✅（依赖像素/域名） |
| L2 | Customer List Custom Audience | 注册用户、付费用户 | 高质量种子、召回 | ✅（最可迁移） |
| L3 | Lookalike（基于 L2/L1） | 1%/2%/5% | 冷启动获客加速 | ✅（推荐核心打法） |
| L4 | Value-based Lookalike（可选） | 用付费金额/分层做种子 | 提升 ROAS | ⚠️ 对数据质量要求更高 |

> **核心原则**：你的“可迁移资产”优先级是：Customer List > Website 行为 > 兴趣/广泛。

#### 3.5.2 Lookalike 分层（Layering）建议

常见的分层方式：

- **按相似度**：`1%`（最像）→ `2%` → `5%`（更广）
- **按种子质量**：`Payers_30D`（付费）> `Registrations_30D`（注册）> `Visitors_30D`（访问）
- **按地域**：每个国家单独建一套（不要把多国家混一个 LAL，除非你明确要 multi-country）

冷启动推荐组合（优先级从高到低）：

1. `LAL_1%_Payers_30D`
2. `LAL_1%_Registrations_30D`
3. `LAL_2%_Payers_30D`
4. `Broad`（兜底拉量）

#### 3.5.3 Exclusion（排除）是“省钱”的核心手段

为了减少浪费、加速学习，你应该系统性做排除：

- 排除已付费用户（做拉新时）
- 排除近 7D 已注册用户（避免重复买量）
- 排除近 7D 已触达高频人群（防止频控导致质量下滑）

#### 3.5.4 事件体系（Signal Engineering）：先跑通，再升级

冷启动阶段要避免一上来就优化“稀疏事件”（比如付费量太小）。建议按阶段升级：

| 阶段 | 优先优化事件 | 目标 | 备注 |
|------|--------------|------|------|
| Phase 1 | `Lead`/`CompleteRegistration` | 先有量，先让模型学起来 | 事件必须稳定且可复现 |
| Phase 2 | `Purchase` | 拿到真实付费优化 | 量不足容易 Learning Limited |
| Phase 3 | `Purchase` + `value` | ROAS/价值优化 | 强依赖金额字段质量 |

> 如果你是 PWA：请优先把 **Pixel + CAPI** 的事件做“稳定、可去重、可匹配”，再谈人群与优化。

#### 3.5.5 Customer List 的“最小工程实现”建议

你要自动化上传 Customer List（而不是手动上传）时，建议做到：

- **数据规范**：email/phone 做归一化（trim/lowercase/去空格/去符号）
- **Hash**：按 Meta 要求做 SHA256（或让平台自动 hash，但要确认一致性）
- **增量更新**：每天一次通常足够；高频更新对学习收益不一定线性
- **口径稳定**：`Payers_30D`、`High_LTV_180D` 这些人群定义不要频繁变

### 3.6 跨账号复用：把“冷启动资产”拆成可迁移与不可迁移

| 资产 | 是否能迁移 | 迁移方式 | 关键注意 |
|------|-----------|---------|----------|
| Customer List / Lookalike | ✅ | 重新在新账号创建/引用 | 种子数据一致性 + 合规 |
| Website 行为人群 | ⚠️ | 取决于 Pixel/域名是否同一主体可用 | 像素/域名资产是否被连坐 |
| 媒体学习状态 | ❌ | 无法搬家 | 新账号仍需积累样本 |

> 你真正能做的是：让新账号从第一天就吃到“更好的受众 + 更好的事件信号”，从而更快过冷启动。

---

## 三点五、个人开发者最小可行实践（国内主体、无外贸经营范围也能先跑通）

你当前的约束可以理解为：

- 以**个人/小团队**为主
- 主体在国内，可能没有海外主体、也没有“外贸”经营范围
- 目标不是一步到位大规模投放，而是先把“Pixel/CAPI → 事件 → 受众 → Lookalike → 冷启动”跑通

下面是一个**可落地的最小闭环（MVP）**，按“先跑通，再增强”的思路设计。

### 3.5.1 你需要准备的最小资产清单

| 资产 | 最小要求 | 备注 |
|------|----------|------|
| Website/PWA | 有域名 + 可部署 JS + 有转化页面 | 最好有明确的注册/下单完成页 |
| BM（Business Manager） | 一个 BM | 建议不要把高风险测试和主业务混在一个 BM |
| Ad Account | 1 个广告账号 | 冷启动建议从小预算开始 |
| Pixel | 1 个 Pixel | Web 事件采集核心 |
| CAPI 回传服务 | 1 个简单服务端接口 | 先实现 1-2 个事件就够 |
| 数据源（用户列表） | 可选：email/phone | 没有也能先用 Website 受众起步 |

### 3.5.2 第一步：先做 Pixel（浏览器端）把“信号”打出来

最低限度先打 3 个事件：

- `PageView`
- `Lead` 或 `CompleteRegistration`
- `Purchase`（如果你有支付）

原则：事件必须“稳定可复现”。哪怕量小，也要先保证每次都能打到。

### 3.5.3 第二步：再做 CAPI（服务端）把丢失的信号补回来

个人开发者最推荐的做法是：

- **先只做 1 个关键事件的 CAPI**（例如 `CompleteRegistration` 或 `Purchase`）
- Browser Pixel 继续打（覆盖面）
- 服务端 CAPI 做补齐（稳定性）
- 统一 `event_id` 去重

你会在 Events Manager 里看到事件是否正常、以及去重是否生效。

### 3.5.4 第三步：不碰 API 也能先跑的人群方案（避开 App Review 的复杂度）

如果你还没准备好走 Marketing API（Developer App 权限/审核/Token 等），先用“纯 UI 能完成”的路径：

1. **Website Custom Audience**：7D/14D/30D 访问、注册、加购等
2. **Lookalike（基于 Website 受众）**：先做 1% 作为冷启动主力

这条路径的优点是：不依赖你去维护 Developer App 和 token 链路。

### 3.5.5 第四步：如果你有用户列表，再上 Customer List（可迁移资产）

当你有 email/phone（且有合规授权）后再升级：

- 先用 Ads Manager 手动上传 Customer List（先跑通）
- 需求稳定后，再考虑 Marketing API 自动化增量更新

### 3.5.6 冷启动的“最小投放”模板（适合个人开发者）

目标：用最少的钱，尽快拿到足够多的“可学习事件”。

1. 受众：`LAL_1%_Registrations_30D`（或基于 Website 事件的 1%）
2. 优化目标：先 `Lead/CompleteRegistration`
3. 预算：小步快跑，先稳定 48 小时再调整
4. 事件质量：优先保证去重与匹配字段（email/phone hash、ip、ua）

### 3.5.7 你最可能卡住的点（个人开发者高频）

| 卡点 | 现象 | 应对 |
|------|------|------|
| 业务验证/主体可信度 | 受众/权限受限、审核更严 | 先走 UI 路径 + 小额合规投放，逐步建立历史 |
| 域名/落地页合规 | 广告频繁被拒登 | 落地页内容与广告一致，避免强跳转/cloaking |
| 事件丢失 | Pixel 事件少、归因差 | 上 CAPI + 做好 `event_id` 去重 |
| 事件稀疏 | 优化 Purchase 学不动 | 先优化注册/线索，等量起来再切付费 |
| 没有用户列表 | 无法做 Customer List | 先用 Website Custom Audience + Lookalike 起量 |

> 实操上，你不需要一开始就“全套都做”。最小闭环就是：Pixel 打通 + 1 个关键事件 CAPI + Website 受众 + Lookalike 1%。

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

### 5.6.1 Lookalike Audiences（相似受众）

- About Lookalike Audiences
  - https://www.facebook.com/business/help/164749007013531
- Create a Lookalike Audience
  - https://www.facebook.com/business/help/465262276878947

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

### 5.9 Custom Conversions / Offline Conversions / AEM

- About custom conversions for web
  - https://www.facebook.com/business/help/780705975381000
- Custom Conversion（API reference）
  - https://developers.facebook.com/docs/marketing-api/reference/custom-conversion/
- Offline Conversions API
  - https://developers.facebook.com/docs/marketing-api/offline-conversions/
- About Meta's Aggregated Event Measurement
  - https://www.facebook.com/business/help/721422165168355
- Aggregated Event Measurement（App Events guide）
  - https://developers.facebook.com/docs/app-events/guides/aggregated-event-measurement/
- Lookalike Audiences（API reference）
  - https://developers.facebook.com/docs/marketing-api/reference/lookalike-audience/
- Create a Lookalike Audience（API reference）
  - https://developers.facebook.com/docs/marketing-api/reference/audiences/create/