# 广告监测与归因全解析

# Ad Measurement & Attribution Deep Dive

> 深入解析广告监测技术原理（S2S/C2S）、效果数据体系、第三方监测与 MMP 的分工协作，以及实战案例。

---

## 一、广告监测技术原理

### 1.1 曝光监测 (Impression Tracking)

广告曝光监测按数据传输方式分为三种：

#### C2S — Client to Server（客户端直传监测平台）

**原理**：广告在客户端（App/浏览器）展示后，由客户端直接将曝光数据发送到第三方监测平台服务器。

**流程**：

```
用户设备 (Client)                   监测平台 (Server)
    │                                    │
    │  1. 广告平台下发广告素材+监测代码     │
    │  2. 广告在客户端渲染展示             │
    │  3. 触发 Tracking Pixel ──────────►│
    │     (HTTP GET 请求，携带设备ID/      │
    │      广告位ID/时间戳等参数)           │
    │                                    │  4. 记录曝光数据
    │                                    │  5. 返回 1x1 透明像素
```

**C2S 细分**：

| 子类型 | 说明 | 适用场景 |
|--------|------|---------|
| **实时加载监测** | 广告加载后即时展示，同时触发 Pixel 上报 | 信息流广告、Banner |
| **预加载监测** | 广告预先缓存，展示时才触发 Pixel 上报 | 开屏广告、插屏广告 |

**C2S 请求示例**：

```
GET https://tracking.example.com/imp?
    campaign_id=12345
    &creative_id=67890
    &placement_id=feed_01
    &device_id=IDFA_xxxx
    &ip=1.2.3.4
    &ua=Mozilla/5.0...
    &ts=1708272000
    &os=iOS
    &app_id=com.example.app
```

**优点**：数据从客户端直达监测平台，不经过媒体服务器，**防篡改能力强**，是目前最主流的曝光监测方式。

**缺点**：受网络环境影响（弱网/断网可能丢失），需要客户端配合加载监测代码。

---

#### S2S — Server to Server（媒体服务器转发监测平台）

**原理**：广告展示后，曝光数据先上报到媒体/广告平台服务器，再由媒体服务器转发给第三方监测平台。

**流程**：

```
用户设备          媒体广告服务器          监测平台
    │                  │                   │
    │  1. 广告展示      │                   │
    │  2. 上报曝光 ───►│                   │
    │                  │  3. 转发曝光数据 ──►│
    │                  │     (S2S API 调用)  │
    │                  │                   │  4. 记录曝光
    │                  │  ◄── 5. 返回确认 ──│
```

**S2S 请求示例**：

```json
POST https://tracking.example.com/api/v1/impression
Content-Type: application/json

{
  "campaign_id": "12345",
  "creative_id": "67890",
  "placement_id": "feed_01",
  "device_id": "IDFA_xxxx",
  "ip": "1.2.3.4",
  "user_agent": "Mozilla/5.0...",
  "timestamp": 1708272000,
  "media_source": "toutiao"
}
```

**优点**：不依赖客户端网络环境，数据传输稳定可靠；适合 OTT/CTV 等无法直接加载 Pixel 的场景。

**缺点**：数据经过媒体服务器中转，**存在被篡改风险**（媒体可以注水）；监测平台无法直接验证曝光真实性。

---

#### SDK 监测（第三方 SDK 嵌入媒体 App）

**原理**：媒体 App 内集成第三方监测平台的 SDK，SDK 自动监听广告展示并直接上报数据。

**流程**：

```
媒体 App (内嵌监测SDK)              监测平台
    │                                │
    │  1. 广告展示                    │
    │  2. 监测SDK自动捕获展示事件      │
    │  3. SDK直接上报 ──────────────►│
    │     (携带设备信息/广告信息/       │
    │      可见性数据/环境数据)         │
    │                                │  4. 记录+分析
```

**优点**：数据最丰富（可采集可见性、停留时长、屏幕位置等）；防篡改能力最强；支持离线广告监测。

**缺点**：需要媒体配合集成 SDK，接入成本高；国内 MMA 推出统一 SDK 规范降低接入成本。

---

#### 三种曝光监测方式对比

| 维度 | C2S (Pixel) | S2S (API) | SDK |
|------|-------------|-----------|-----|
| **数据路径** | 客户端→监测平台 | 客户端→媒体→监测平台 | 客户端(SDK)→监测平台 |
| **防篡改** | 强 | 弱（经媒体中转） | 最强 |
| **数据丰富度** | 中 | 低 | 高（可见性/停留时长） |
| **接入成本** | 低（加监测链接） | 中（API 对接） | 高（集成 SDK） |
| **网络依赖** | 依赖客户端网络 | 不依赖 | 依赖客户端网络 |
| **适用场景** | App/Web 广告（主流） | OTT/CTV/程序化 | 品牌广告/头部媒体 |
| **主流程度** | ⭐⭐⭐ 最主流 | ⭐⭐ | ⭐⭐ 品牌广告主流 |

---

### 1.2 点击监测 (Click Tracking)

#### 同步监测（302 重定向）

**原理**：用户点击广告后，先跳转到监测平台的链接，监测平台记录点击数据后通过 302 重定向将用户引导到最终落地页。

```
用户点击广告
    │
    ▼
监测平台链接 (https://track.example.com/click?...)
    │
    │  记录点击数据（设备ID/IP/时间戳/广告信息）
    │
    ▼  302 Redirect
落地页 (https://landing.advertiser.com/...)
```

**优点**：监测平台能直接获取用户设备信息（IP/UA），数据准确。

**缺点**：增加一次跳转延迟（~100-300ms）；不支持回传 IDFA 等设备级参数（部分场景）。

---

#### 异步监测（异步上报）

**原理**：用户点击广告后直接跳转到落地页，同时客户端异步将点击数据上报给监测平台。

```
用户点击广告
    │
    ├──► 直接跳转到落地页（无延迟）
    │
    └──► 异步上报点击数据到监测平台
         (后台 HTTP 请求，不影响用户体验)
```

**优点**：不影响用户跳转体验，无额外延迟。

**缺点**：异步请求可能因网络问题丢失。

---

#### 点击监测方式对比

| 维度 | 同步（302 跳转） | 异步上报 |
|------|-----------------|---------|
| **用户体验** | 有跳转延迟 | 无感知 |
| **数据准确性** | 高（直接获取设备信息） | 中（可能丢失） |
| **适用场景** | Web 广告、落地页跳转 | App 内广告、信息流 |
| **主流程度** | Web 端主流 | App 端主流 |

---

## 二、媒体/DSP 效果数据体系

### 2.1 数据层级结构

媒体和 DSP 的效果数据按**层级**组织，从粗到细：

```
账户 (Account)
 └── 广告计划 (Campaign)           ← 预算/投放目标/排期
      └── 广告组/单元 (Ad Set/Ad Group)  ← 定向/出价/版位
           └── 广告创意 (Ad/Creative)     ← 素材/文案/落地页
```

### 2.2 核心效果指标

每个层级都可以查看以下指标，通常按**分钟级/小时级/天级**聚合：

#### 基础量级指标

| 指标 | 英文 | 公式 | 说明 |
|------|------|------|------|
| **曝光量** | Impressions | — | 广告被展示的次数 |
| **点击数** | Clicks | — | 广告被点击的次数 |
| **转化数** | Conversions | — | 完成目标行为的次数（安装/注册/付费） |

#### 效率指标

| 指标 | 英文 | 公式 | 说明 |
|------|------|------|------|
| **点击率** | CTR | Clicks / Impressions × 100% | 衡量创意吸引力 |
| **转化率** | CVR | Conversions / Clicks × 100% | 衡量落地页/产品转化能力 |

#### 成本指标

| 指标 | 英文 | 公式 | 说明 |
|------|------|------|------|
| **千次展示成本** | CPM | Cost / Impressions × 1000 | 曝光成本 |
| **点击成本** | CPC | Cost / Clicks | 获取一次点击的成本 |
| **转化成本** | CPA | Cost / Conversions | 获取一次转化的成本 |

#### 收益指标

| 指标 | 英文 | 公式 | 说明 |
|------|------|------|------|
| **广告支出回报率** | ROAS | Revenue / Cost × 100% | 每花 1 元广告费赚回多少 |
| **投资回报率** | ROI | (Revenue - Cost) / Cost × 100% | 扣除成本后的回报 |

### 2.3 数据报表示例

#### 广告计划 (Campaign) 级别 — 小时聚合报表

| 时间 | 计划名称 | 曝光量 | 点击数 | 转化数 | CTR | CVR | 花费(¥) | CPC | CPA | ROAS |
|------|---------|--------|--------|--------|-----|-----|---------|-----|-----|------|
| 10:00 | 游戏拉新_iOS | 52,300 | 1,046 | 31 | 2.0% | 3.0% | 3,138 | 3.0 | 101.2 | 285% |
| 10:00 | 游戏拉新_Android | 78,500 | 2,041 | 82 | 2.6% | 4.0% | 4,082 | 2.0 | 49.8 | 420% |
| 11:00 | 游戏拉新_iOS | 48,100 | 913 | 27 | 1.9% | 3.0% | 2,739 | 3.0 | 101.4 | 278% |

#### 广告创意 (Creative) 级别 — 天聚合报表

| 日期 | 创意ID | 素材类型 | 曝光量 | 点击数 | CTR | 转化数 | CVR | CPA | ROAS |
|------|--------|---------|--------|--------|-----|--------|-----|-----|------|
| 02-18 | cr_001 | 竖版视频 | 320,000 | 8,960 | 2.8% | 358 | 4.0% | 44.7 | 450% |
| 02-18 | cr_002 | 横版视频 | 280,000 | 5,600 | 2.0% | 168 | 3.0% | 83.3 | 240% |
| 02-18 | cr_003 | 图文 | 150,000 | 2,250 | 1.5% | 45 | 2.0% | 166.7 | 120% |

### 2.4 数据来源与差异

**同一个广告投放，不同数据源的数据会有差异**：

| 数据源 | 曝光量 | 点击数 | 转化数 | 说明 |
|--------|--------|--------|--------|------|
| **媒体后台** (如巨量引擎) | 1,000,000 | 25,000 | 800 | 媒体自己的统计 |
| **第三方监测** (如秒针) | 920,000 | 23,500 | — | 过滤无效流量后 |
| **MMP** (如 AppsFlyer) | — | 22,000 | 680 | 去重归因后 |
| **广告主自有** (BI 系统) | — | — | 650 | 自有数据库统计 |

差异原因：统计口径不同、反作弊过滤规则不同、归因窗口不同、数据延迟。行业惯例：差异率控制在 **±5-10%** 以内为正常。

---

## 三、第三方监测 vs MMP 详解

### 3.1 第三方广告监测 (Ad Verification)

**定义**：独立于广告买卖双方的第三方机构，验证广告投放的真实性、可见性、品牌安全。

**代表公司**：

| 公司 | 市场 | 核心能力 | 官方文档 |
|------|------|---------|---------|
| **IAS** (Integral Ad Science) | 海外 | 可见性/品牌安全/IVT | [ias.com/products](https://integralads.com/products/) |
| **DoubleVerify** (DV) | 海外 | 可见性/品牌安全/注意力指标 | [doubleverify.com](https://doubleverify.com/) |
| **MOAT** (Oracle) | 海外 | 可见性/注意力 | [oracle.com/moat](https://www.oracle.com/advertising/moat/) |
| **秒针系统** (Miaozhen) | 国内 | 曝光验证/触达去重/品牌安全 | [miaozhen.com](https://www.miaozhen.com/) |
| **尼尔森** (Nielsen) | 全球 | 触达/GRP/跨屏 | [nielsen.com](https://www.nielsen.com/) |

**核心能力**：

| 能力 | 说明 |
|------|------|
| **广告可见性 (Viewability)** | 广告是否真正被用户看到（MRC 标准：50% 像素可见 ≥1 秒；视频 50% 可见 ≥2 秒） |
| **品牌安全 (Brand Safety)** | 广告是否出现在不当内容旁边（暴力/色情/政治/虚假新闻） |
| **无效流量 (IVT)** | 检测机器人、爬虫、数据中心流量等虚假流量 |
| **曝光验证 (Impression Verification)** | 独立计数曝光，与媒体数据交叉验证 |
| **触达与频次 (Reach & Frequency)** | 去重后的独立用户触达和频次统计 |
| **注意力指标 (Attention Metrics)** | 用户实际关注广告的时长和程度（新兴指标） |

---

### 3.2 MMP (Mobile Measurement Partner)

**定义**：移动广告归因和效果衡量的第三方平台，核心解决"用户从哪个广告渠道来"的问题。

**代表公司**：

| 公司 | 市场 | 核心优势 | 官方文档 |
|------|------|---------|---------|
| **AppsFlyer** | 海外（全球 #1） | 归因/Protect360 反作弊/ROI 分析 | [support.appsflyer.com](https://support.appsflyer.com/) |
| **Adjust** | 海外 | 归因/反作弊/Audience Builder | [help.adjust.com](https://help.adjust.com/) |
| **Branch** | 海外 | 深度链接/归因 | [docs.branch.io](https://docs.branch.io/) |
| **Singular** | 海外 | 归因/成本聚合/ROI | [support.singular.net](https://support.singular.net/) |
| **Kochava** | 海外 | 归因/反作弊/隐私合规 | [support.kochava.com](https://support.kochava.com/) |
| **热云数据** (TrackingIO) | 国内 | 归因/反作弊/国内媒体覆盖 | [reyun.com](https://www.reyun.com/) |
| **TalkingData** | 国内 | 归因/数据分析 | [talkingdata.com](https://www.talkingdata.com/) |
| **openinstall** | 国内 | 免填邀请码/渠道统计 | [openinstall.io](https://www.openinstall.io/) |

**核心能力**：

| 能力 | 说明 |
|------|------|
| **安装归因 (Install Attribution)** | 判断 App 安装来自哪个广告渠道（最后点击/最后触点） |
| **深度事件归因 (In-App Event)** | 注册、付费、留存等后续行为归因到渠道 |
| **反作弊 (Fraud Protection)** | 检测安装劫持、点击注入、SDK 欺骗、设备农场 |
| **深度链接 (Deep Linking)** | 广告→App 内指定页面的无缝跳转 |
| **SKAdNetwork 管理** | 管理 iOS SKAN 归因（转化值映射/聚合报告） |
| **跨渠道报表** | 统一口径的多渠道效果对比（CPA/ROAS/LTV） |
| **成本聚合 (Cost Aggregation)** | 自动拉取各媒体的花费数据，计算真实 ROI |
| **Audience 管理** | 基于归因数据创建人群包，回传给媒体做再营销 |

---

### 3.3 分工与协作

**第三方监测和 MMP 解决不同层面的问题，是互补关系**：

```
广告投放全链路:

  广告展示 ─────────────────────────────────────► 用户转化 ──► 深度行为
     │                                               │           │
     │◄─── 第三方监测的领域 ───►│                      │           │
     │ 曝光验证/可见性/品牌安全/IVT                     │           │
     │                         │                      │           │
     │                         │◄──── MMP 的领域 ──────────────────►│
     │                         │ 点击归因/安装归因/深度事件/LTV/反作弊
```

| 维度 | 第三方监测 | MMP |
|------|-----------|-----|
| **核心问题** | 广告是否真实有效地展示了？ | 用户从哪个渠道来的？ |
| **关注阶段** | 曝光侧（展示质量） | 转化侧（归因效果） |
| **主要客户** | 品牌广告主为主 | 效果广告主 / App 开发者 |
| **数据来源** | 广告展示端（Pixel/SDK 验证） | 点击/安装/事件端（SDK 归因） |
| **行业标准** | MRC 认证 | 各媒体官方认证 Partner |
| **收费模式** | 按 CPM 收费 | 按归因量/套餐收费 |

**协作场景**：

- **品牌广告主**：第三方监测验证曝光质量 + MMP 追踪后续转化
- **效果广告主**：MMP 做归因 + 第三方监测辅助验证流量质量
- **全链路衡量**：第三方监测确认"广告被看到了" → MMP 确认"用户因此转化了"

---

### 3.4 重叠与区别

| 重叠领域 | 第三方监测 | MMP |
|---------|-----------|-----|
| **反作弊** | ✅ 检测无效流量 (IVT) — 展示端 | ✅ 检测安装欺诈 — 转化端 |
| **曝光计数** | ✅ 独立验证曝光数 | ✅ 记录曝光（用于归因） |
| **基础报表** | ✅ 曝光/点击 | ✅ 曝光/点击/转化/收入 |

**第三方监测独有**：可见性 (Viewability)、品牌安全、跨媒体触达去重、GRP/TRP。

**MMP 独有**：精确归因、深度事件追踪 (LTV)、SKAdNetwork 管理、深度链接、跨渠道 ROI 对比、成本聚合。

---

## 四、为什么不用 DSP/媒体的监测和归因数据？

### 4.1 利益冲突 — "既当运动员又当裁判"

DSP/媒体既是广告**售卖方**又是数据**提供方**，天然利益冲突：

- 媒体有动机**高报曝光数**（多收钱）
- DSP 有动机**高报转化数**（证明效果好）
- 媒体有动机**低报无效流量**（承认作弊 = 退款）

### 4.2 数据口径不一致

| 问题 | 示例 |
|------|------|
| **曝光定义不同** | A 平台：服务端发出即算；B 平台：客户端渲染才算 |
| **点击定义不同** | 有的算所有点击，有的只算有效点击 |
| **归因窗口不同** | Meta 7天点击+1天曝光，Google 30天，TikTok 7天 |
| **归因模型不同** | 有的用最后点击，有的用最后触点 |
| **反作弊标准不同** | 各平台过滤规则不同 |

广告主在 3 个平台投放，拿到 3 套口径不同的数据，无法直接对比。

### 4.3 归因抢功 — 同一转化被多次计算

```
用户路径: 看了 TikTok 广告 → Google 搜索 → 点了 Meta 广告 → 安装 App

TikTok 报告: 这个安装是我的！（曝光归因）
Google 报告: 这个安装是我的！（搜索归因）
Meta 报告:   这个安装是我的！（最后点击归因）

结果: 1 个安装被 3 个平台各报了 1 次 = 虚增 200%
```

MMP 作为独立裁判，用统一归因逻辑判定归属，**去重**，确保 1 个转化只归因给 1 个渠道。

### 4.4 跨平台视角缺失

每个媒体只能看到自己平台的数据，没有人能看到用户的完整转化路径。MMP 集成所有渠道数据，提供**全局视角**。

### 4.5 行业信任与合规

- **MRC 认证**：第三方监测需通过 MRC 审计认证
- **行业标准**：IAB 推荐使用独立第三方验证
- **合同要求**：大品牌广告主合同中要求必须使用第三方监测
- **审计需求**：上市公司需要独立第三方数据支持财务审计

---

## 五、实战案例：海外广告主使用 AppsFlyer 投放 Meta + Google

### 5.1 背景

假设一家中国游戏公司（如米哈游）要在海外推广一款新游戏，选择 **Meta Ads** 和 **Google Ads** 作为主要投放渠道，使用 **AppsFlyer** 作为 MMP。

### 5.2 前期准备

#### Step 1：集成 AppsFlyer SDK

```
开发团队在 App 中集成 AppsFlyer SDK：

iOS:  pod 'AppsFlyerFramework'
Android:  implementation 'com.appsflyer:af-android-sdk:6.x.x'

初始化代码：
  - 设置 Dev Key（AppsFlyer 后台获取）
  - 设置 App ID
  - 启动 SDK
  - 注册深度事件（注册/付费/通关等）
```

#### Step 2：在 AppsFlyer 后台配置 Meta Ads

1. 登录 AppsFlyer → **Collaborate** → **Active Integrations** → 选择 **Meta Ads**
2. **Integration Tab**：
   - 开启 **Activate Partner**
   - 输入 Meta **App ID**
   - 配置 **Install Referrer Decryption Key**（Android，从 Meta Developer 后台获取）
   - 设置归因窗口：**Click-through 7 天 / View-through 1 天**（匹配 Meta 默认窗口）
3. **In-App Events Tab**：
   - 映射 AppsFlyer 事件到 Meta 事件：
     - `af_complete_registration` → `CompleteRegistration`
     - `af_purchase` → `Purchase`
     - `af_level_achieved` → `AchieveLevel`
   - 开启 **Send Revenue** 回传收入数据
4. **Cost Tab**：开启成本数据拉取（需 Meta Business Manager 授权）

#### Step 3：在 AppsFlyer 后台配置 Google Ads

1. 登录 AppsFlyer → **Collaborate** → **Active Integrations** → 选择 **Google Ads**
2. 在 Google Ads 后台：**Tools & Settings** → **Linked Accounts** → 选择 **AppsFlyer** → 生成 **Link ID**
3. 在 AppsFlyer 中输入 **Link ID**
4. 映射 In-App Events：
   - `af_complete_registration` → `first_open` (或自定义)
   - `af_purchase` → `in_app_purchase`
5. 设置归因窗口：**Click-through 30 天 / View-through 1 天**（匹配 Google 默认）
6. 开启 Cost 数据拉取

### 5.3 投放期间的数据流

```
                    ┌──────────────┐
                    │  用户看到广告  │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     Meta Ads 展示广告           Google Ads 展示广告
              │                         │
              │  曝光/点击数据            │  曝光/点击数据
              │  (Meta 自报告)           │  (Google 自报告)
              ▼                         ▼
     ┌────────────────────────────────────────┐
     │           AppsFlyer SDK (App 内)        │
     │                                        │
     │  1. 记录安装事件                         │
     │  2. 收集设备信息 (IDFA/GAID)            │
     │  3. 向 AppsFlyer 服务器发送安装回调       │
     │  4. 记录深度事件 (注册/付费/留存)         │
     └────────────────┬───────────────────────┘
                      │
                      ▼
     ┌────────────────────────────────────────┐
     │         AppsFlyer 归因服务器             │
     │                                        │
     │  1. 收到安装回调                         │
     │  2. 查询 Meta/Google 的点击/曝光记录     │
     │  3. 执行归因逻辑（最后点击优先）          │
     │  4. 判定归因渠道                         │
     │  5. 向 Meta/Google 发送 Postback        │
     │     (告知转化归因结果，用于优化)          │
     └────────────────┬───────────────────────┘
                      │
                      ▼
     ┌────────────────────────────────────────┐
     │         AppsFlyer Dashboard             │
     │                                        │
     │  统一报表：                              │
     │  - Meta: 5,000 安装, CPA $2.5, ROAS 320%│
     │  - Google: 3,200 安装, CPA $3.1, ROAS 280%│
     │  - Organic: 1,800 安装                   │
     │  - 总计: 10,000 安装 (去重后)             │
     └────────────────────────────────────────┘
```

### 5.4 归因决策过程（具体示例）

```
用户 A 的行为路径：

Day 1: 在 Instagram (Meta) 看到游戏视频广告 → 没点击
Day 3: 在 YouTube (Google) 看到游戏广告 → 点击了
Day 5: 在 Google Search 搜索游戏名 → 点击广告
Day 5: 下载并安装游戏

AppsFlyer 归因决策：
  ├── Meta 曝光归因候选：Day 1 曝光（在 1 天 View-through 窗口外）→ ❌ 过期
  ├── Google 点击归因候选 1：Day 3 点击（在 30 天窗口内）→ ✅ 有效
  ├── Google 点击归因候选 2：Day 5 点击（在 30 天窗口内）→ ✅ 有效
  │
  └── 最后点击归因 → 归因给 Google Search (Day 5 点击)

结果：
  - AppsFlyer 报告：归因给 Google Ads
  - Meta 自己的报告：可能也会报告这个安装（因为有曝光）
  - 这就是为什么需要 MMP 做去重归因
```

### 5.5 Postback 回传机制

AppsFlyer 归因完成后，会向对应媒体发送 **Postback**（转化回传），用于媒体的算法优化：

```
AppsFlyer → Meta Postback:
POST https://www.facebook.com/tr/
{
  "event_name": "MOBILE_APP_INSTALL",
  "advertiser_tracking_enabled": 1,
  "application_tracking_enabled": 1,
  "extinfo": ["i2", ...],
  "custom_events": [
    {"_eventName": "af_purchase", "_valueToSum": 9.99, "fb_currency": "USD"}
  ]
}

AppsFlyer → Google Postback:
通过 Google Ads API 回传转化数据，
Google 用此数据优化 UAC (Universal App Campaign) 的出价和定向。
```

### 5.6 日常运营看板

投放经理每天在 AppsFlyer Dashboard 查看：

| 渠道 | 花费($) | 安装数 | CPI | 注册数 | 注册成本 | 付费用户 | 付费率 | ROAS D7 |
|------|---------|--------|-----|--------|---------|---------|--------|---------|
| Meta - Instagram | 12,500 | 5,000 | 2.50 | 3,500 | 3.57 | 250 | 5.0% | 320% |
| Meta - Facebook | 8,000 | 2,800 | 2.86 | 1,960 | 4.08 | 168 | 6.0% | 290% |
| Google - UAC | 10,000 | 3,200 | 3.13 | 2,240 | 4.46 | 192 | 6.0% | 280% |
| Google - Search | 3,000 | 800 | 3.75 | 640 | 4.69 | 72 | 9.0% | 350% |
| **总计** | **33,500** | **11,800** | **2.84** | **8,340** | **4.02** | **682** | **5.8%** | **305%** |

**决策**：Google Search ROAS 最高 (350%) 但量小 → 适当追加预算；Meta Instagram 量大且 ROAS 不错 → 主力渠道；Facebook 和 Google UAC 表现中等 → 优化创意和定向。

---

## 六、总结对比表

| 维度 | 第三方监测 | MMP | DSP/媒体自有数据 |
|------|-----------|-----|-----------------|
| **独立性** | ✅ 独立第三方 | ✅ 独立第三方 | ❌ 利益相关方 |
| **核心价值** | 曝光质量验证 | 转化归因 | 投放执行 |
| **可见性** | ✅ | ❌ | ❌ |
| **品牌安全** | ✅ | ❌ | 部分 |
| **归因** | ❌ | ✅ | ✅ 但有偏差 |
| **深度事件/LTV** | ❌ | ✅ | 部分 |
| **跨渠道对比** | 部分（触达去重） | ✅（效果/ROI） | ❌ |
| **统一口径** | ✅ | ✅ | ❌ |
| **反作弊** | ✅ 展示端 | ✅ 转化端 | 有但标准不一 |
| **成本** | 按 CPM 收费 | 按归因量/套餐 | 免费 |

---

## 七、面试回答要点

**如果被问到"第三方监测和 MMP 的区别"：**

> 第三方监测和 MMP 是广告效果衡量体系中互补的两个角色。
>
> **第三方监测**（如 IAS、秒针）解决"广告是否真实有效地展示了"，关注曝光可见性、品牌安全、无效流量检测，通过 C2S/S2S/SDK 三种方式采集曝光数据，服务品牌广告主为主。
>
> **MMP**（如 AppsFlyer、Adjust）解决"用户从哪个渠道来"，关注安装归因、深度事件追踪、跨渠道 ROI 对比，通过 SDK 集成到广告主 App 中采集转化数据，服务效果广告主和 App 开发者为主。
>
> 两者在反作弊领域有重叠但侧重不同：第三方监测侧重展示端（虚假曝光），MMP 侧重转化端（安装欺诈）。
>
> 不能只用 DSP/媒体自有数据的核心原因：利益冲突（既当运动员又当裁判）、数据口径不一致（各平台定义不同无法对比）、归因抢功（同一转化被多个平台重复计算）、以及缺乏跨平台全局视角。

**English version:**

> Third-party measurement (IAS, DoubleVerify) and MMPs (AppsFlyer, Adjust) are complementary in ad measurement.
>
> **Third-party measurement** verifies "was the ad actually seen?" — focusing on viewability, brand safety, and IVT detection via C2S/S2S/SDK tracking, primarily serving brand advertisers.
>
> **MMPs** solve "which channel drove the conversion?" — focusing on attribution, in-app events, and cross-channel ROI via SDK integration, primarily serving performance advertisers.
>
> They overlap in anti-fraud but with different focus: measurement on the impression side, MMPs on the conversion side.
>
> We can't rely solely on DSP/media data because of: conflict of interest, inconsistent metrics, attribution stealing (same conversion counted multiple times), and lack of cross-platform visibility.

---

## 八、AppsFlyer 文档与 API 详解

### 8.1 文档体系

AppsFlyer 有两套文档入口：

| 文档站 | URL | 面向人群 | 内容 |
|--------|-----|---------|------|
| **Help Center** | [support.appsflyer.com](https://support.appsflyer.com/) | 产品经理/运营/投放 | 功能说明、媒体对接指南、Dashboard 使用 |
| **Dev Hub** | [dev.appsflyer.com](https://dev.appsflyer.com/) | 开发者/工程师 | SDK 集成、API Reference、技术文档 |

### 8.2 SDK 集成文档

| 平台 | 文档 | 说明 |
|------|------|------|
| **Android SDK** | [dev.appsflyer.com/hc/docs/android-sdk](https://dev.appsflyer.com/hc/docs/integrate-android-sdk) | Gradle 集成、初始化、事件上报 |
| **iOS SDK** | [dev.appsflyer.com/hc/docs/ios-sdk](https://dev.appsflyer.com/hc/docs/integrate-ios-sdk) | CocoaPods/SPM 集成、ATT 处理 |
| **Unity SDK** | [dev.appsflyer.com/hc/docs/unity-sdk](https://dev.appsflyer.com/hc/docs/unity-plugin) | 游戏引擎集成 |
| **React Native** | [dev.appsflyer.com/hc/docs/react-native](https://dev.appsflyer.com/hc/docs/react-native-plugin) | 跨平台框架 |
| **Flutter** | [dev.appsflyer.com/hc/docs/flutter](https://dev.appsflyer.com/hc/docs/flutter-plugin) | 跨平台框架 |
| **SDK 集成概览** | [support.appsflyer.com/...207032126](https://support.appsflyer.com/hc/en-us/articles/207032126-SDK-integration-overview) | 集成规划清单 |

### 8.3 API Reference（开发者核心）

AppsFlyer 提供丰富的 API，按功能分类：

#### 归因与监测 API

| API | 端点 | 说明 |
|-----|------|------|
| **Click Engagement API** | `GET/POST /click/app/{platform}/{app-id}` | 发送点击数据用于归因 |
| **Impression Engagement API** | `GET/POST /impression/app/{platform}/{app-id}` | 发送曝光数据用于 VTA 归因 |
| **S2S Events API (Mobile)** | `POST /inappevent/{app-id}` | 服务端上报 App 内事件（注册/付费等） |
| **Web S2S API** | `POST /v1.0/s2s/web/event` | Web 端服务端事件上报 |
| **PC/Console/CTV API** | `POST /first-open/app/{platform}/{app-id}` | PC/主机/CTV 设备事件 |
| **Preload Measurement API** | `POST /app/{app-id}` | 预装归因 |
| **Preload C2S API** | `POST /app/{app-id}` | 预装 C2S 归因 |

#### 数据拉取 API

| API | 说明 |
|-----|------|
| **Raw Data Pull API V2** | 拉取原始数据（安装/事件/卸载等明细） |
| **Aggregate Pull API V2** | 拉取聚合报表数据（按渠道/日期/地区等维度） |
| **Cohort API** | 拉取留存/LTV 等队列分析数据 |
| **Master API** | 拉取汇总级别的 KPI 数据 |
| **ROI360 Net Revenue API** | 拉取净收入和 ROI 数据 |
| **InCost API** | 拉取各渠道花费数据 |

#### 管理与配置 API

| API | 说明 |
|-----|------|
| **OneLink API v2.0** | 创建/管理深度链接 |
| **Partner Integration Settings API** | 配置媒体合作伙伴集成 |
| **SKAN CV Schema API** | 配置 SKAdNetwork 转化值映射 |
| **SKAN Aggregated Postback API** | 拉取 SKAN 聚合回传数据 |
| **App Management API v2.0** | 管理 App 配置 |
| **Audience External API** | 创建/管理人群包 |

**API 文档入口**：[dev.appsflyer.com/hc/reference/api-reference-overview](https://dev.appsflyer.com/hc/reference/api-reference-overview)

### 8.4 媒体对接文档

| 媒体 | AppsFlyer 对接文档 | 关键配置 |
|------|-------------------|---------|
| **Meta Ads** | [Meta ads integration setup](https://support.appsflyer.com/hc/en-us/articles/207033826) | App ID、Install Referrer Key、事件映射、Cost API |
| **Google Ads** | [Google Ads integration setup](https://support.appsflyer.com/hc/en-us/articles/115002504686) | Link ID、事件映射、归因窗口 |
| **TikTok for Business** | [TikTok integration](https://support.appsflyer.com/hc/en-us/articles/360002187258) | 事件映射、SRN 配置 |
| **Apple Search Ads** | [Apple Search Ads integration](https://support.appsflyer.com/hc/en-us/articles/115003100866) | API Credentials、Campaign Group |
| **Snap Ads** | [Snap Ads integration](https://support.appsflyer.com/hc/en-us/articles/360001430768) | Snap App ID |
| **Unity Ads** | [Unity Ads integration](https://support.appsflyer.com/hc/en-us/articles/360001429497) | Game ID |

### 8.5 归因模型文档

| 文档 | 链接 |
|------|------|
| **归因模型总览** | [AppsFlyer attribution model](https://support.appsflyer.com/hc/en-us/articles/207447053) |
| **归因窗口设置** | [Attribution windows](https://support.appsflyer.com/hc/en-us/articles/207447053#attribution-lookback-window) |
| **SRN 归因（Meta/Google 等）** | [Self-reporting networks](https://support.appsflyer.com/hc/en-us/articles/207447053#self-reporting-networks) |
| **Protect360 反作弊** | [Protect360 overview](https://support.appsflyer.com/hc/en-us/articles/207447053#protect360) |

---

## 九、AppsFlyer MCP Server（AI 集成）

### 9.1 什么是 MCP？

**MCP (Model Context Protocol)** 是一个开放标准协议，允许 AI 助手（如 Claude、Cursor、Windsurf）直接连接外部数据源和工具。通过 MCP，你可以让 AI 直接查询 AppsFlyer 的归因数据、生成报表、分析投放效果。

### 9.2 AppsFlyer 官方 MCP Server

AppsFlyer 提供了**官方 MCP Server**，支持远程连接：

| 属性 | 值 |
|------|-----|
| **名称** | `com.appsflyer/mcp` |
| **远程 URL** | `https://mcp.appsflyer.com` |
| **认证方式** | OAuth |
| **传输协议** | Streamable HTTP |
| **费用** | Free Tier 可用 |
| **产品页面** | [appsflyer.com/products/mcp](https://www.appsflyer.com/products/mcp/) |

### 9.3 社区版 MCP Server（开源）

GitHub 上有开源的 AppsFlyer MCP Server，可本地部署：

**仓库**：[github.com/ysntony/appsflyer-mcp](https://github.com/ysntony/appsflyer-mcp)

**安装**：

```bash
git clone https://github.com/ysntony/appsflyer-mcp
cd appsflyer-mcp
uv sync
```

**配置环境变量**：

```bash
export APPSFLYER_API_BASE_URL="https://hq1.appsflyer.com"
export APPSFLYER_TOKEN="your_api_token_here"
```

**MCP 配置（用于 Cursor/Windsurf/Claude Desktop 等）**：

```json
{
  "mcpServers": {
    "appsflyer": {
      "command": "uv",
      "args": ["run", "python", "run_server.py"],
      "cwd": "/path/to/appsflyer-mcp",
      "env": {
        "APPSFLYER_API_BASE_URL": "https://hq1.appsflyer.com",
        "APPSFLYER_TOKEN": "your_api_token_here"
      }
    }
  }
}
```

**可用工具 (Tools)**：

| Tool | 说明 |
|------|------|
| `get_aggregate_data` | 从 AppsFlyer Pull API 拉取聚合报表数据 |
| `test_appsflyer_connection` | 测试 AppsFlyer API 连接 |

**支持的报表类型**：

| 报表类型 | 说明 |
|---------|------|
| `partners_report` | 渠道合作伙伴效果数据 |
| `partners_by_date_report` | 按日期的渠道效果数据 |
| `daily_report` | 每日聚合数据（默认） |
| `geo_report` | 地理位置效果数据 |
| `geo_by_date_report` | 按日期的地理位置数据 |

**使用示例**（在 AI 助手中）：

```
用户: 帮我查看过去7天 Meta Ads 在美国的安装数据

AI 调用 MCP → get_aggregate_data:
  app_id: "com.example.game"
  report_type: "geo_by_date_report"
  from_date: "2024-02-11"
  to_date: "2024-02-18"
  
返回: 按日期×地区的安装数/收入/CPI 等聚合数据
```

### 9.4 其他 MMP 的 MCP 状态

| MMP | MCP 状态 | 说明 |
|-----|---------|------|
| **AppsFlyer** | ✅ 官方 + 社区 | 官方远程 MCP + GitHub 开源版 |
| **Adjust** | ❌ 暂无 | 无官方或社区 MCP，需通过 API 自行对接 |
| **Branch** | ❌ 暂无 | 无 MCP |
| **Singular** | ✅ 社区 | 有社区版 Singular Reporting MCP |
| **热云数据** | ❌ 暂无 | 无 MCP，需通过 API 对接 |

---

## 十、Adjust 与热云数据文档

### 10.1 Adjust 文档体系

| 文档站 | URL | 说明 |
|--------|-----|------|
| **Help Center** | [help.adjust.com](https://help.adjust.com/) | 产品功能、媒体对接、Dashboard |
| **Dev Hub** | [dev.adjust.com](https://dev.adjust.com/) | SDK 集成、API 文档 |

**核心文档**：

| 文档 | 说明 |
|------|------|
| [Android SDK 集成](https://dev.adjust.com/en/sdk/android) | Android 集成指南 |
| [iOS SDK 集成](https://dev.adjust.com/en/sdk/ios) | iOS 集成指南 |
| [S2S 事件 API](https://dev.adjust.com/en/api/s2s-api) | 服务端事件上报 |
| [KPI Service API](https://dev.adjust.com/en/api/kpi-service) | 拉取聚合 KPI 数据 |
| [Raw Data Export](https://dev.adjust.com/en/api/raw-data-export) | 原始数据导出 |
| [Fraud Prevention](https://help.adjust.com/en/article/fraud-prevention) | 反作弊功能 |
| [归因方法](https://help.adjust.com/en/article/attribution-methods) | 归因模型说明 |
| [S2S 概念说明](https://www.adjust.com/glossary/server-to-server-s2s/) | S2S 技术概念 |

### 10.2 热云数据 (TrackingIO) 文档体系

| 文档站 | URL | 说明 |
|--------|-----|------|
| **官网** | [reyun.com](https://www.reyun.com/) | 产品介绍 |
| **开发者文档** | [docs.trackingio.com](http://docs.trackingio.com/) | SDK/API 技术文档 |

**核心文档**：

| 文档 | 说明 |
|------|------|
| [Android SDK 集成](http://docs.trackingio.com/android_sdk.html) | Android 集成指南 |
| [iOS SDK 集成](http://docs.trackingio.com/ios_sdk.html) | iOS 集成指南 |
| [服务端 API](http://docs.trackingio.com/server_api.html) | S2S 事件上报 |
| [渠道对接](http://docs.trackingio.com/) | 国内主流媒体（巨量/腾讯/快手等）对接 |

**热云特色**：深度覆盖国内媒体生态（巨量引擎、腾讯广告、快手、百度、OPPO/vivo/小米应用商店等），支持 OAID（国内 Android 设备标识）。

---

## 十一、行业标准与规范

| 标准 | 链接 | 说明 |
|------|------|------|
| **MRC 可见性标准** | [mediaratingcouncil.org](http://mediaratingcouncil.org/) | 曝光可见性定义（50% 像素 ≥1 秒） |
| **IAB OpenRTB** | [iabtechlab.com](https://iabtechlab.com/) | 程序化广告竞价协议 |
| **IAB IVT 指南** | [iabtechlab.com](https://iabtechlab.com/) | 无效流量检测标准 (GIVT/SIVT) |
| **MMA 中国** | [mmachina.cn](http://www.mmachina.cn/) | 国内移动广告监测标准/统一 SDK 规范 |
| **CAID** | — | 中国广告标识符（替代 IDFA） |
| **OAID** | [msa-alliance.cn](http://www.msa-alliance.cn/) | 移动安全联盟匿名设备标识 |
