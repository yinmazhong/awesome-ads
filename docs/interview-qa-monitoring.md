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

#### 真实痛点：广告账号封禁与冷启动压力

出海广告主（尤其是游戏/电商）面临一个高频痛点：**Meta/Google 广告账号容易被封**。

常见封号原因：
- 创意违规（夸大宣传、敏感内容）
- 账号信任度低（新账号、支付异常）
- 大规模跑量触发风控
- 落地页合规问题

封号后果：**账号历史数据（受众、像素事件、优化信号）全部清零**，新账号需要从零开始积累转化数据，算法才能重新学习和优化，这个过程就是**冷启动**。

冷启动期间 CPA 高、ROAS 低，通常需要 50-200 个转化事件后算法才能稳定，这段时间花费大、效果差，是出海广告主最头疼的阶段。

---

#### 使用 AppsFlyer 对冷启动的影响：优劣势分析

**✅ 优势：AppsFlyer 帮助加速冷启动**

| 优势 | 说明 |
|------|------|
| **归因数据独立于媒体账号** | AppsFlyer 的 SDK 和归因数据存在广告主侧，账号被封不影响历史归因数据 |
| **快速切换新账号** | 新 Meta/Google 账号创建后，在 AppsFlyer 后台重新配置 Integration 即可，SDK 不需要改动 |
| **Postback 持续回传** | 新账号上线后，AppsFlyer 立即开始向新账号回传转化事件，帮助算法快速积累信号 |
| **历史受众数据可迁移** | AppsFlyer 的 Audiences 功能可将历史用户人群包（付费用户、高价值用户）导出并上传到新账号，跳过冷启动的受众学习阶段 |
| **跨账号/跨渠道归因不中断** | 即使某个账号被封，其他渠道（Google、TikTok）的归因和数据采集不受影响，整体投放视图完整 |
| **Protect360 反作弊持续运行** | 不依赖媒体账号，封号期间反作弊保护不中断 |

**⚠️ 劣势：AppsFlyer 无法解决的问题**

| 劣势 | 说明 |
|------|------|
| **媒体账号的像素/转化历史无法迁移** | Meta Pixel 的历史事件数据绑定在 Business Manager，新账号的 Pixel 是空的，算法仍需重新学习 |
| **新账号的账号信任度从零开始** | AppsFlyer 无法提升新账号在 Meta/Google 的信任评分，审核更严、限额更低 |
| **冷启动期间花费仍然较高** | AppsFlyer 加速的是"有转化数据后"的优化速度，但冷启动初期（前 50 个转化）的高 CPA 无法完全避免 |
| **iOS ATT 限制依然存在** | 新账号同样面临 IDFA 不可用的问题，SKAdNetwork 的聚合延迟无法绕过 |

**实际操作中的应对策略**：

```
封号 → 立即行动：

1. 保留 AppsFlyer 历史数据（不受影响）
2. 用 AppsFlyer Audiences 导出高价值用户人群包
3. 注册新 Meta BM → 创建新广告账号 → 新 Pixel
4. 在 AppsFlyer 后台更新 Meta Integration（换新账号 App ID）
5. 将人群包上传到新账号作为种子受众（Lookalike 基础）
6. 新账号上线 → AppsFlyer 立即开始回传转化 → 算法加速学习
7. 目标：48-72 小时内恢复投放，而不是从零等待 1-2 周
```

#### 重点：上一个被封禁账号积累的“归因/转化数据”如何复用到新账号？

这里要先把“数据”拆成两类，否则很容易误解：

| 数据资产 | 存在哪里 | 封号后是否还在 | 能否直接迁移到新账号 |
|---------|---------|---------------|----------------------|
| **A. 媒体账号侧的投放学习（Learning）** | Meta/Google 的广告账号/像素/转化模型 | ❌ 基本清零 | ❌ 不能直接迁移 |
| **B. 广告主侧的归因与用户资产** | AppsFlyer + 自己的用户库/数据仓库 | ✅ 完整保留 | ✅ 可以复用 |

换句话说：**AppsFlyer 能帮你把“广告主侧的历史数据”复用出来，但无法把“被封账号在媒体侧的学习状态”复制过去。**

---

##### 1) AppsFlyer 侧：复用“用户资产”和“效果知识”

**你能复用的核心资产**：

- **历史归因的 Raw Data**：你可以拿到设备级明细（安装/注册/付费），作为新账号冷启动时的基线（CPI/CPA/ROAS、国家/机型/渠道结构）。
- **高价值用户人群**：从 AppsFlyer 里按规则圈人（例如 D7 ROAS>1、付费用户、特定国家留存高的用户）。

**怎么复用到新账号**（最常用两条线）：

1. **AppsFlyer Audiences → 导出/同步人群包**
   - 人群例子：
     - `Payers_Last_30D`（近30天付费用户）
     - `High_LTV_US`（美国高 LTV 用户）
     - `Churned_Payers`（流失付费用户，用于召回）
   - 用途：
     - 上传到新广告账号作为 **Custom Audience**
     - 基于 Custom Audience 做 **Lookalike**，让新账号从“高质量种子”开始找人，减少纯随机探索成本

2. **保持事件口径一致（对新账号最关键）**
   - 确保 AppsFlyer 里 `af_complete_registration`、`af_purchase` 等事件定义、币种、收入字段保持一致
   - 确保 Meta/Google 的事件映射（event mapping）在新账号上也按同样口径配置
   - 这样新账号一上线，媒体收到的 Postback 信号就与旧账号一致，模型能更快收敛

---

##### 2) Meta 侧：能复用什么？不能复用什么？

**不能复用的（现实约束）**：

- **广告账号级学习状态无法迁移**：新广告账号/新广告组仍会进入 Learning Phase。
- **Pixel 历史与账号绑定**：如果封的是 Business Manager 或像素相关资产，新像素几乎等于从零开始。

**能复用的（实操抓手）**：

- **受众资产可以复用**：把旧账号（或 AppsFlyer）沉淀的高价值用户做成 Custom Audience/Lookalike。
- **事件信号可以快速恢复**：新账号通过 AppsFlyer Postback 继续收到设备级转化（install/register/purchase/value）。

**让新账号“快进入稳定期”的关键动作**：

- **优先用“价值更强”的事件做优化目标**：从 `install` 快速切到 `registration` / `purchase` / `value`（但要保证量够，否则会 Learning Limited）。
- **用 S2S 付费事件提高信号质量**：服务端校验后再上报 `af_purchase`，减少假事件对模型的污染。
- **前 48 小时保证稳定的转化流**：预算不要频繁大幅调整，尽快累计到可学习的样本量。

---

##### 3) Google 侧：复用逻辑（更偏“点击级标识”）

Google 的转化回传里，最关键的是 **`gclid`（点击标识）**：

- 旧账号的 `gclid` 只对应旧账号的点击，无法迁移。
- 但新账号上线后，AppsFlyer 会继续把新点击产生的 `gclid` 关联到安装/事件，并通过 Google 的转化接口回传。
- 你能复用的是：
  - 老账号沉淀出来的地域/素材/关键词结构
  - 以及 AppsFlyer 的历史数据基线与人群包（用于再营销/相似人群策略）

---

##### 4) 一句话总结（回答面试/业务复盘都好用）

> 封号后，媒体侧的“学习”基本无法搬家；但 AppsFlyer 把归因数据、事件口径和高价值人群沉淀在广告主侧，让你可以快速在新账号恢复同口径的 Postback 信号，并用人群包作为种子受众，把冷启动从“完全随机探索”变成“带先验的快速收敛”。

---

#### 广告主开发者需要做哪些事？

使用 AppsFlyer 涉及两类人：**投放运营**（配置后台）和**开发者**（集成代码）。以下聚焦开发者职责：

**一次性工作（App 上线前）**

| 任务 | 说明 | 难度 |
|------|------|------|
| **集成 AppsFlyer SDK** | iOS/Android 各自接入，约 1-2 天工作量 | ⭐⭐ |
| **初始化 SDK** | 设置 Dev Key、App ID，启动 SDK | ⭐ |
| **上报标准事件** | 在注册/付费/关键行为节点调用 `logEvent()` | ⭐⭐ |
| **处理 Deep Link** | 配置 OneLink，处理广告落地页跳转到 App 内指定页面 | ⭐⭐⭐ |
| **iOS ATT 弹窗** | 在合适时机请求 ATT 权限，影响 IDFA 获取率 | ⭐⭐ |
| **测试归因** | 用 AppsFlyer 测试设备验证安装归因是否正常 | ⭐ |

**封号后切换账号时（开发者几乎不需要动）**

这是 AppsFlyer 的核心优势之一：**封号换账号，开发者不需要改代码**。

```
封号前：SDK 上报 → AppsFlyer → Postback 到 Meta 账号 A
封号后：SDK 上报 → AppsFlyer → Postback 到 Meta 账号 B（只需运营在后台改配置）
```

开发者唯一可能需要做的：如果新账号使用了不同的 Deep Link 落地页，需要更新 OneLink 配置。

**需要服务端配合的场景（S2S 事件）**

如果付费事件发生在服务端（如虚拟货币充值、订阅续费），需要后端开发者额外接入 **S2S Events API**：

```bash
POST https://api2.appsflyer.com/inappevent/{app-id}
Authorization: {dev-key}
Content-Type: application/json

{
  "appsflyer_id": "1234567890123-1234567",   ← SDK 生成的设备唯一 ID
  "eventName": "af_purchase",
  "eventValue": "{\"af_revenue\":9.99,\"af_currency\":\"USD\"}",
  "eventTime": "2024-02-19 10:30:00.000"
}
```

> 为什么需要 S2S？客户端 SDK 上报的付费事件可能被刷单（假充值），服务端验证后再上报更可靠，也是 Meta/Google 算法优化的更高质量信号。

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

### 5.5 Postback 回传机制与数据颗粒度

AppsFlyer 归因完成后，会向对应媒体发送 **Postback**（转化回传），用于媒体算法优化。**这是整个链路中最关键的环节之一**，颗粒度直接决定媒体算法能优化到什么程度。

#### Postback 的数据颗粒度

Postback 是**设备级（Device-level）**的，即每一个归因事件都单独回传一条记录，而不是聚合后的汇总数据。

```
用户 A 安装 → AppsFlyer 归因到 Meta → 立即发送 1 条 Postback 给 Meta
用户 B 安装 → AppsFlyer 归因到 Google → 立即发送 1 条 Postback 给 Google
用户 A 付费 → AppsFlyer 记录事件 → 发送 1 条付费 Postback 给 Meta
```

**Meta Postback 示例（设备级，含设备标识）**：

```http
POST https://graph.facebook.com/v17.0/{pixel_id}/events
Content-Type: application/json

{
  "data": [{
    "event_name": "Purchase",
    "event_time": 1708300800,
    "event_id": "af_event_abc123",
    "user_data": {
      "madid": "GAID_or_IDFA_here",      ← 设备广告标识符（IDFA/GAID）
      "client_ip_address": "1.2.3.4",
      "client_user_agent": "Mozilla/5.0..."
    },
    "custom_data": {
      "currency": "USD",
      "value": 9.99,                      ← 付费金额
      "content_ids": ["item_001"]
    },
    "app_data": {
      "advertiser_tracking_enabled": 1,
      "application_tracking_enabled": 1,
      "campaign_ids": "23847382910"       ← Meta 广告系列 ID
    }
  }],
  "access_token": "..."
}
```

**Google Ads Postback 示例（通过 Google Ads API 回传）**：

```
AppsFlyer → Google Ads Conversion API:
- gclid: "Cj0KCQiA..."         ← Google Click ID（点击时生成，标识具体广告点击）
- conversion_action: "purchase"
- conversion_value: 9.99
- conversion_currency: "USD"
- conversion_date_time: "2024-02-19 10:30:00+00:00"
```

#### Postback 颗粒度对媒体算法的影响

| 回传内容 | 媒体能做什么 |
|---------|------------|
| **设备 ID (IDFA/GAID)** | 精准匹配到具体广告展示/点击记录，确认归因 |
| **事件类型（安装/注册/付费）** | 区分不同转化目标，分别优化 |
| **事件价值（付费金额）** | 优化 ROAS 出价（Value-based Bidding） |
| **事件时间戳** | 计算归因窗口内的转化，排除窗口外的 |
| **Campaign/AdSet ID** | 定位到具体广告组，精准调整出价 |

> **关键点**：Meta 的 Advantage+ 和 Google 的 Smart Bidding 都依赖这些设备级 Postback 来训练模型。如果只回传聚合数据（如"今天共 100 个安装"），算法无法知道哪条具体广告有效，优化效果会大幅下降。

#### iOS ATT 对 Postback 颗粒度的影响

iOS 14.5+ 后，用户可拒绝追踪（ATT），导致 IDFA 不可用：

| 用户状态 | Postback 颗粒度 |
|---------|----------------|
| **用户同意 ATT** | 完整设备级 Postback，含 IDFA |
| **用户拒绝 ATT** | 通过 SKAdNetwork 回传，**聚合级**，无设备 ID，有延迟（24-48h），有转化值上限 |
| **Android 用户** | GAID 默认可用（除非用户手动关闭），设备级 Postback 正常 |

---

### 5.6 AppsFlyer 中能看到什么数据？明细 vs 聚合

这是一个非常重要的问题：**接入 SDK 后，你能在 AppsFlyer 看到/拉取到的数据是什么粒度的？**

#### 结论：两种数据都有，但访问方式不同

| 数据类型 | 粒度 | 访问方式 | 包含设备号？ |
|---------|------|---------|------------|
| **聚合报表** | 渠道/日期/地区维度汇总 | Dashboard + Aggregate Pull API | ❌ 无设备 ID |
| **原始数据（Raw Data）** | 每条安装/事件一行记录 | Raw Data Pull API / Push API / Data Locker | ✅ 含设备 ID |

#### 聚合数据（你在 Dashboard 看到的）

Dashboard 默认展示的是**聚合维度**的数据，例如：

```
渠道: Meta Ads
日期: 2024-02-19
安装数: 5,000
注册数: 3,500
付费用户: 250
收入: $2,490
CPI: $2.50
ROAS: 320%
```

这是按 Campaign → AdSet → Creative 层级聚合的，**看不到单个用户/设备的信息**。

#### 原始明细数据（Raw Data）

通过 **Raw Data Pull API** 或 **Push API** 可以拉取每一条安装/事件的明细记录：

```csv
install_time,         media_source, campaign,          adset,        advertising_id,              idfa,                                device_model, country, event_name,  revenue
2024-02-19 10:23:11, Meta Ads,     Game_iOS_US_Feb,   Lookalike_1,  ,                            A1B2C3D4-E5F6-...(IDFA),             iPhone 15,    US,      install,     0
2024-02-19 10:45:33, Meta Ads,     Game_iOS_US_Feb,   Lookalike_1,  ,                            A1B2C3D4-E5F6-...(IDFA),             iPhone 15,    US,      af_purchase, 9.99
2024-02-19 11:02:44, Google Ads,   UAC_Android_US,    ,             38f7a2b1-...(GAID),           ,                                    Pixel 8,      US,      install,     0
```

**每一行就是一个设备的一个事件**，包含：
- `advertising_id`：Android 设备的 GAID（Google Advertising ID）
- `idfa`：iOS 设备的 IDFA（需用户同意 ATT）
- `install_time`、`event_name`、`revenue` 等完整字段

#### 设备号粒度数据的访问权限

| 数据 | 谁能看 | 限制 |
|------|--------|------|
| **广告主自己的 Raw Data** | ✅ 广告主完全可以拉取 | 通过 Raw Data Pull API，需要 API Token |
| **媒体（Meta/Google）能看到的** | ❌ 媒体看不到 AppsFlyer 的原始数据 | 媒体只收到 Postback，不能访问 AppsFlyer 后台 |
| **iOS 拒绝 ATT 的用户** | ⚠️ IDFA 字段为空 | 只有 SKAdNetwork 聚合数据 |
| **Android 用户** | ✅ GAID 通常可用 | 用户可手动关闭 |

#### 你的理解是否正确？

> "看上去只需要用户在 App 上接入 SDK，然后就可以从 AppsFlyer 看完整数据了"

**基本正确，但有几点补充**：

1. ✅ **接入 SDK 是核心**：SDK 负责收集设备信息、发送安装/事件回调给 AppsFlyer 服务器
2. ✅ **可以看到完整数据**：包括聚合报表和设备级原始数据（Raw Data）
3. ⚠️ **还需要配置媒体集成**：在 AppsFlyer 后台配置 Meta/Google 的对接，才能让 AppsFlyer 知道点击来自哪个广告，完成归因
4. ⚠️ **iOS 有 ATT 限制**：拒绝追踪的用户，IDFA 为空，归因走 SKAdNetwork（聚合，无设备 ID）
5. ✅ **Raw Data 有设备号粒度**：通过 API 可以拉取每个设备的安装/事件明细，含 IDFA/GAID

---

### 5.7 场景扩展：产品不做 App，只做 PWA（且 Meta 账号不绑定 App）怎么办？

你补充的限制是：**我们的 Meta 投放账号中暂时不添加绑定 App**。这会直接改变链路中“归因与优化信号”的核心载体：

- 在 **App 场景**，媒体优化主要吃的是 App Install / App Event 信号（通过 MMP SDK + Postback）。
- 在 **PWA 场景**（且 Meta 账号不绑定 App），媒体优化只能吃 **Web 信号**（Pixel + Conversions API 等）。

因此结论变化如下：

| 目标 | 是否还能复用“上一个账号积累的数据” | 复用方式 |
|------|-------------------------------|---------|
| **复用用户资产（种子受众）** | ✅ 仍然可以 | Customer List / Custom Audience / Lookalike（来自你自己的用户库或历史网站用户） |
| **复用 AppsFlyer 的 App 归因资产** | ⚠️ 基本不适用 | 没有 App/SDK 就没有标准 App 归因与设备级安装链路 |
| **复用优化信号（让算法快速收敛）** | ✅ 仍然可以，但载体变了 | 从“AppsFlyer Postback”切换为“Pixel + CAPI（服务端回传）” |
| **复用媒体侧 learning 状态** | ❌ 仍然不行 | 新账号仍会重新 Learning |

#### 文档依据：为什么“只做 PWA + Meta 不绑定 App”会削弱 AppsFlyer 作为 App MMP 的杠杆？

这个判断来自 AppsFlyer 官方文档对“App 集成”和“Web 集成”的明确分层：

- **Meta Ads（App）集成是以 App 为核心前提**：需要在 AppsFlyer 集成流程中配置 App 相关信息（如 Meta 的 App ID、事件映射等）。
  - https://support.appsflyer.com/hc/en-us/articles/207033826-Meta-ads-integration-setup

- **Web 场景在 AppsFlyer 里属于 PBA（People-Based Attribution）体系**：通过 Web SDK 记录 web visits/web events，并可用 Web-S2S 补全服务端事件。
  - https://support.appsflyer.com/hc/en-us/articles/360001610038-PBA-Web-SDK-integration-guide
  - https://support.appsflyer.com/hc/en-us/articles/360006997298-Web-Server-to-server-events-API-for-PBA-Web-S2S

- **AppsFlyer 的 API Reference 也把 Web 与 App 分为不同类目**（例如 Web Server-TO-Server API 与 Mobile S2S 分开列出）。
  - https://dev.appsflyer.com/hc/reference/api-reference-overview

因此，当你的产品只有 PWA 且 Meta 账号不绑定 App 时，业务会自然从“App MMP（SDK+install+in-app events+postback）”迁移到“Web 信号（Pixel/CAPI + Web 事件）”，AppsFlyer 的价值更多体现为 Web/PBA 或数据方法论，而不是经典 App MMP 主链路。

#### 在该限制下，最推荐的落地方案（Meta）

如果不能绑定 App，你要把“冷启动加速”寄托在两件事上：**人群复用** + **高质量 Web 转化回传**。

**1) 人群复用（最能跨账号迁移的资产）**

- 从你自己的用户库构建：注册用户、付费用户、高 LTV 用户
- 上传到新账号：
  - Custom Audience（自定义受众）
  - Lookalike Audience（相似受众）

**2) Web 转化回传（让新账号尽快积累“可学习样本”）**

- 在 PWA 中埋点 **Meta Pixel**（浏览器端）
- 同时接入 **Meta Conversions API (CAPI)**（服务端）做“同口径双写/去重”
  - Browser Pixel 用于覆盖面
  - Server CAPI 用于稳定性与抗丢（尤其是 Safari/隐私限制下）

建议回传事件层级：

| 阶段 | 优先事件 | 说明 |
|------|----------|------|
| 冷启动前期 | `Lead` / `CompleteRegistration` | 量大，便于先把模型跑通 |
| 稳定后 | `Purchase` + `value` | 做价值优化（ROAS/Value） |

关键工程要点：

- **事件去重**：Pixel 与 CAPI 必须使用同一个 `event_id` 做 dedup，否则会重复记账影响优化
- **身份匹配**：尽可能提供可匹配字段（如 email/phone hash、ip、ua），提升归因匹配率
- **口径稳定**：事件命名、币种、金额字段保持一致，封号切换账号后才能“快速恢复同口径信号”

> 在这个限制下，AppsFlyer 的价值主要从“移动端 MMP 归因”转为“数据方法论参考”。真正的执行系统会变成 Pixel/CAPI + 自己的数据仓库。

#### 如果你希望“像 App 一样”做归因：PWA → TWA（Android APK）备选

如果后续业务允许，你可以把 PWA 用 **TWA/Bubblewrap** 打包为 Android APK：

- 这时你就“有 App 了”，可接入 AppsFlyer Android SDK
- 可以拿到更像 App 的归因/事件明细（GAID 等），以及更完整的 MMP 能力
- 但注意：你当前的限制（Meta 账号不绑定 App）仍然意味着 **Meta 侧还是主要吃 Web 信号**，除非限制解除

#### Google 侧（PWA）

Google 对 Web 的优化链路更成熟：

- 用 gtag/GA4 记录 Web 转化
- 通过 Google Ads 的 Web 转化/增强型转化/离线转化导入（按你的链路选择）回传
- 同样用“人群 + 稳定转化信号”缩短新账号学习时间

### 5.8 日常运营看板

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

AppsFlyer 提供了**官方 MCP Server**（目前为 Closed Beta），支持远程连接：

| 属性 | 值 |
|------|-----|
| **名称** | `com.appsflyer/mcp` |
| **远程 URL** | `https://mcp.appsflyer.com` |
| **认证方式** | OAuth（连接时自动发起 OAuth 授权流程，无需手动配置 API Key） |
| **传输协议** | Streamable HTTP |
| **费用** | Free Tier (Zero Plan) 可用 |
| **产品页面** | [appsflyer.com/products/mcp](https://www.appsflyer.com/products/mcp/) |
| **官方文档** | [AppsFlyer MCP Help Center](https://support.appsflyer.com/hc/en-us/articles/36349070304785) |

#### 在 Windsurf 中配置 AppsFlyer 官方 MCP

> **注意**：AppsFlyer 官方 MCP 目前处于 **Closed Beta** 阶段，Windsurf 内置 Marketplace 中可能尚未上线。推荐使用手动配置方式。

**手动配置步骤（当前最可靠方式）**

**第一步：打开 mcp_config.json**

在 Windsurf 中按 `Cmd+Shift+P`（Mac）→ 搜索 `Open MCP Config` → 回车，打开配置文件。

或直接在终端编辑：

```bash
open ~/.codeium/windsurf/mcp_config.json
```

**第二步：添加 AppsFlyer 配置**

```json
{
  "mcpServers": {
    "appsflyer": {
      "serverUrl": "https://mcp.appsflyer.com/mcp"
    }
  }
}
```

> 如果文件已有其他 MCP 配置，在 `mcpServers` 对象里追加 `"appsflyer": {...}` 这一项即可，注意 JSON 逗号分隔。

**第三步：保存并重启 Cascade**

保存文件后，在 Cascade 面板点击 **Refresh** 或重启 Windsurf，AppsFlyer MCP 会出现在工具列表中。

**第四步：OAuth 授权（首次使用时）**

当 Cascade 第一次调用 AppsFlyer 工具时，会自动弹出浏览器跳转到 AppsFlyer 登录页。用你的 AppsFlyer 账号登录并点击授权，之后 Windsurf 自动保存 Token，后续无需重复操作。

---

**备选：如果已有 AppsFlyer API V2 Token（跳过 OAuth）**

在 AppsFlyer 后台 → **Account Settings** → **API Tokens** 获取 V2 Token，然后配置：

```json
{
  "mcpServers": {
    "appsflyer": {
      "serverUrl": "https://mcp.appsflyer.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:APPSFLYER_TOKEN}"
      }
    }
  }
}
```

在 `~/.zshrc`（或 `~/.bash_profile`）中添加：

```bash
export APPSFLYER_TOKEN="your_appsflyer_api_v2_token"
```

然后 `source ~/.zshrc` 使其生效。

---

#### 个人开发者注册账号使用 MCP

**核心结论：MCP 的 OAuth 就是登录你自己的 AppsFlyer 账号，不是注册独立 OAuth App。**

你需要先有一个 AppsFlyer 账号，才能完成 OAuth 授权。

**注册方式：AppsFlyer Zero Plan（免费）**

| 属性 | 说明 |
|------|------|
| **费用** | $0 |
| **归因额度** | 每月最多 12,000 次归因 |
| **注册要求** | 填写邮箱即可注册，App 可以后续再添加（不强制要求已上架） |
| **注册地址** | [appsflyer.com/sign-up](https://www.appsflyer.com/sign-up/) |

注册后你会得到一个 AppsFlyer 账号，可以：
- 在 Dashboard 里添加一个 App（哪怕是测试 App）
- 获取 Dev Key 和 API Token
- 完成 MCP OAuth 授权

**个人开发者的实际场景分析**

| 目标 | 是否需要 MCP | 建议 |
|------|------------|------|
| **学习 AppsFlyer 文档和 API** | ❌ 不需要 | 文档完全公开，直接访问 [dev.appsflyer.com](https://dev.appsflyer.com/) |
| **有自己的 App，想接入归因** | ✅ 有价值 | 注册 Zero Plan，接入 SDK，MCP 可查询自己的数据 |
| **研究 MCP 技术本身** | ⚠️ 可用社区版 | 用 [github.com/ysntony/appsflyer-mcp](https://github.com/ysntony/appsflyer-mcp) + 自己的 API Token，无需 OAuth |
| **没有 App，纯学习研究** | ❌ 意义不大 | MCP 返回的是你账号下的数据，没有投放数据时返回空 |

**💡 当前阶段建议（个人开发者无 App）：**

1. **学习文档**：直接读 [dev.appsflyer.com](https://dev.appsflyer.com/) 和 [support.appsflyer.com](https://support.appsflyer.com/)，无需账号
2. **体验 MCP**：注册免费账号 → 用社区版 MCP（API Token 方式，比 OAuth 更简单）
3. **有 App 后**：接入 SDK → 产生真实数据 → 官方 MCP 才真正有价值

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
