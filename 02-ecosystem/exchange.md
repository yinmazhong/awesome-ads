# 广告交易平台 (Ad Exchange & RTB)

## 一句话概述

Ad Exchange 是连接 DSP 和 SSP 的程序化广告交易市场，通过 RTB (实时竞价) 协议在毫秒级完成广告的买卖撮合。

---

## 交易架构全景

```
广告主侧                    交易层                     媒体侧
┌────────┐              ┌──────────┐              ┌────────┐
│  DSP-1 │──┐           │          │           ┌──│ SSP-1  │
│  DSP-2 │──┤  Bid Req  │    Ad    │  Ad Req   ├──│ SSP-2  │
│  DSP-3 │──┼──────────►│ Exchange │◄──────────┼──│ SSP-3  │
│  DSP-4 │──┤  Bid Resp │          │  Ad Resp  ├──│ SSP-4  │
│  DSP-5 │──┘           │          │           └──│ SSP-5  │
└────────┘              └──────────┘              └────────┘
                              │
                        ┌─────┴─────┐
                        │   DMP     │
                        │ (数据增强) │
                        └───────────┘
```

---

## Ad Exchange (广告交易平台)

### 定义
类似股票交易所，Ad Exchange 是广告库存的交易市场，通过技术手段实现广告的自动化买卖。

### 核心职责

| 职责 | 说明 |
|------|------|
| **撮合交易** | 连接买方 (DSP) 和卖方 (SSP)，完成竞价 |
| **竞价管理** | 执行竞价逻辑，选出胜出者 |
| **协议标准** | 定义 Bid Request / Bid Response 格式 |
| **质量控制** | 过滤无效流量、违规广告 |
| **结算清算** | 记录交易，处理计费 |

### 主要 Ad Exchange

| 平台 | 说明 |
|------|------|
| **Google AdX** | 全球最大，Google Ad Manager 内置 |
| **OpenX** | 独立 Ad Exchange |
| **Xandr (AppNexus)** | 微软旗下 |
| **巨量引擎 ADX** | 字节跳动，对接穿山甲流量 |
| **腾讯 ADX** | 对接优量汇流量 |
| **百度 BES** | 百度广告交易平台 |

---

## RTB (Real-Time Bidding) 完整流程

### 时序图

```
用户        媒体App/Web      SSP/AdX         DSP-1    DSP-2    DSP-3
 │              │               │              │        │        │
 │──访问页面──►│               │              │        │        │
 │              │──Ad Request──►│              │        │        │
 │              │               │──Bid Req────►│        │        │
 │              │               │──Bid Req───────────►│        │
 │              │               │──Bid Req──────────────────►│
 │              │               │              │        │        │
 │              │               │◄──Bid ¥30────│        │        │
 │              │               │◄──Bid ¥45──────────│        │
 │              │               │◄──No Bid─────────────────│
 │              │               │              │        │        │
 │              │               │  竞价: DSP-2 胜出 (¥45)    │
 │              │               │              │        │        │
 │              │◄──Ad Resp─────│              │        │        │
 │◄──展示广告──│               │              │        │        │
 │              │               │              │        │        │
 │──点击广告──►│──Click Track──►│──Win Notice──────────►│        │
```

### 详细步骤

#### Step 1: 广告请求 (Ad Request)
用户访问页面/App → 客户端向 SSP/Ad Server 发送广告请求

```json
// 请求包含的信息
{
  "user": {
    "device_id": "xxx",
    "ip": "1.2.3.4",
    "ua": "Mozilla/5.0...",
    "geo": {"country": "CN", "city": "Shanghai"}
  },
  "ad_slot": {
    "id": "slot_001",
    "size": "640x100",
    "type": "banner",
    "app": {"name": "NewsApp", "bundle": "com.news.app"}
  }
}
```

#### Step 2: 竞价请求 (Bid Request)
SSP/AdX 向已接入的 DSP 发送竞价请求 (基于 OpenRTB 协议)

```json
// OpenRTB Bid Request (简化)
{
  "id": "auction_12345",
  "imp": [{
    "id": "1",
    "banner": {"w": 640, "h": 100},
    "bidfloor": 5.0,
    "bidfloorcur": "CNY"
  }],
  "device": {
    "ifa": "device_id_xxx",
    "ip": "1.2.3.4",
    "os": "Android",
    "model": "Pixel 6"
  },
  "user": {
    "id": "user_hash_xxx"
  }
}
```

#### Step 3: 竞价响应 (Bid Response)
DSP 在 ~100ms 内返回出价

```json
// OpenRTB Bid Response (简化)
{
  "id": "auction_12345",
  "seatbid": [{
    "bid": [{
      "id": "bid_001",
      "impid": "1",
      "price": 45.0,
      "adm": "<div>广告创意HTML</div>",
      "nurl": "https://dsp.com/win?price=${AUCTION_PRICE}",
      "crid": "creative_001",
      "adomain": ["advertiser.com"]
    }]
  }]
}
```

#### Step 4: 竞价决策 (Auction)
Ad Exchange 执行竞价逻辑，选出胜出者

#### Step 5: 广告展示 + 胜出通知 (Win Notice)
广告展示给用户，同时通知胜出 DSP 实际成交价

---

## OpenRTB 协议

### 定义
IAB (Interactive Advertising Bureau) 制定的 RTB 标准协议，定义了 Bid Request 和 Bid Response 的数据格式。

### 版本演进

| 版本 | 年份 | 特点 |
|------|------|------|
| **OpenRTB 2.0** | 2012 | 基础版本 |
| **OpenRTB 2.5** | 2016 | 增加原生广告、视频支持 |
| **OpenRTB 2.6** | 2022 | 增加 CTV、DOOH 支持 |
| **OpenRTB 3.0** | 2022 | 全新架构，分层设计 |

### Bid Request 核心字段

| 对象 | 说明 | 关键字段 |
|------|------|---------|
| **BidRequest** | 顶层对象 | id, imp[], device, user, app/site |
| **Imp** | 广告位信息 | id, banner/video/native, bidfloor |
| **Device** | 设备信息 | ifa, ip, ua, os, model, geo |
| **User** | 用户信息 | id, buyeruid, gender, yob |
| **App/Site** | 媒体信息 | name, bundle, domain, cat[] |

### Bid Response 核心字段

| 对象 | 说明 | 关键字段 |
|------|------|---------|
| **BidResponse** | 顶层对象 | id, seatbid[] |
| **SeatBid** | 买方席位 | bid[], seat |
| **Bid** | 出价信息 | id, impid, price, adm, nurl, crid |

---

## 竞价机制详解

### 第二价格竞价 (Second-Price Auction / GSP)

```
DSP-A 出价: ¥50
DSP-B 出价: ¥45
DSP-C 出价: ¥30

胜出者: DSP-A
实际支付: ¥45 + ¥0.01 = ¥45.01 (第二高价 + 最小增量)
```

- **优点**: 鼓励真实出价 (Truthful Bidding)
- **缺点**: 在 Header Bidding 多层竞价下失效

### 第一价格竞价 (First-Price Auction)

```
DSP-A 出价: ¥50
DSP-B 出价: ¥45

胜出者: DSP-A
实际支付: ¥50 (自己的出价)
```

- **优点**: 简单透明，适合多层竞价
- **缺点**: 需要 Bid Shading (出价折扣) 策略
- **行业趋势**: 2019 年后成为主流

### Bid Shading (出价折扣)

在第一价格竞价下，DSP 不会按真实估值出价，而是打折：

```
真实估值: ¥50
Bid Shading 后出价: ¥35–¥40

Shading 策略:
- 基于历史成交价分布
- 基于竞争激烈程度
- 基于流量质量
```

### VCG 机制 (Vickrey-Clarke-Groves)

- 基于边际贡献计费
- 理论上最优，但计算复杂
- Facebook 曾使用，后转向第一价格

---

## 交易类型对比

| 交易类型 | 价格 | 库存 | 自动化 | 适用场景 |
|---------|------|------|--------|---------|
| **直接购买 (IO)** | 固定 | 保量 | 手动 | 品牌广告、独占资源 |
| **PG (程序化保量)** | 固定 | 保量 | 自动 | 品牌广告程序化执行 |
| **PMP (私有市场)** | 底价+竞价 | 优先 | 自动 | 优质流量定向售卖 |
| **RTB (公开竞价)** | 竞价 | 不保量 | 自动 | 效果广告大规模投放 |

### 优先级 (Programmatic Waterfall)

```
优先级从高到低:
1. 直接销售 (Sponsorship / Direct)
2. 程序化保量 (PG)
3. 私有市场 (PMP)
4. 公开竞价 (Open RTB)
5. 兜底广告 (House Ads / Backfill)
```

---

## 技术挑战

### 延迟要求
- 整个 RTB 流程: < 100ms
- DSP 响应超时: 通常 50–80ms
- 网络传输: ~10ms
- Ad Exchange 处理: ~10ms

### 高并发
- 头部 Ad Exchange QPS: 百万级
- 每次请求发送给 5–20 个 DSP
- 总 QPS 可达千万级

### 数据量
- 每天数十亿次竞价请求
- 日志量: TB 级别
- 需要高效的日志采集和处理系统

---

## 国内 vs 海外交易生态

### 海外 (开放生态)

```
广告主 → 独立DSP → 多个Ad Exchange → 多个SSP → 多个媒体
                         ↕
                    独立DMP
```

- 各环节独立，互相竞争
- OpenRTB 标准广泛采用
- 透明度较高

### 国内 (围墙花园)

```
广告主 → 巨量引擎 ──────────────────→ 抖音/头条/穿山甲
广告主 → 腾讯广告 ──────────────────→ 微信/QQ/优量汇
广告主 → 百度营销 ──────────────────→ 百度搜索/百青藤
```

- 大媒体自建闭环，数据不互通
- 独立 Ad Exchange 空间小
- RTA (Real-Time API) 作为替代方案

### RTA (Real-Time API)

```
媒体广告系统 ──RTA请求──► 广告主服务器
                              │
                         广告主实时决策:
                         - 是否参竞
                         - 出价调整
                         - 人群筛选
                              │
媒体广告系统 ◄──RTA响应──┘
```

- 国内特色: 广告主通过 RTA 接口参与媒体的广告决策
- 弥补了国内缺乏开放 RTB 生态的不足
- 主要平台: 巨量引擎、腾讯广告均支持 RTA

---

## 与大数据开发的关联

- **竞价日志处理**: 海量 Bid Request/Response 日志的采集和分析
- **实时数据管道**: RTB 事件的实时处理 (曝光/点击/转化)
- **数据对账**: DSP 与 AdX 之间的数据一致性校验
- **流量分析**: 竞价参与率、胜出率、成交价分布分析
- **RTA 数据链路**: RTA 请求/响应的实时处理和特征服务
- **OpenRTB 日志解析**: Protobuf/JSON 格式的日志解析和入库

---

## 面试高频问题

1. RTB 的完整流程是什么？从用户访问到广告展示经历了哪些步骤？
2. 第一价格和第二价格竞价的区别？什么是 Bid Shading？
3. OpenRTB 协议的核心字段有哪些？
4. PMP 和 RTB 的区别是什么？
5. 国内广告交易生态和海外有什么不同？什么是 RTA？

---

## 推荐阅读

- [OpenRTB 2.6 规范](https://iabtechlab.com/standards/openrtb/)
- 《计算广告》第 7 章 — 广告交易
- 《程序化广告》第 3-4 章
- [Google Ad Manager 文档](https://admanager.google.com/)
