# Awesome Ads 🎯

> 在线广告行业知识地图 — 从业者的系统化知识梳理

### 📖 [在线阅读文档 → https://ads.vexa.xin](https://ads.vexa.xin)

---

## 知识地图总览

```
                            ┌─────────────────┐
                            │   在线广告生态    │
                            └────────┬────────┘
            ┌──────────┬──────────┬──┴───┬──────────┬──────────┐
            ▼          ▼          ▼      ▼          ▼          ▼
       ┌────────┐ ┌────────┐ ┌──────┐ ┌──────┐ ┌───────┐ ┌───────┐
       │ 广告主 │ │ 媒体方 │ │ 平台 │ │ 数据 │ │ 算法  │ │ 工程  │
       │  侧    │ │  侧    │ │  侧  │ │  侧  │ │  侧   │ │  侧   │
       └────────┘ └────────┘ └──────┘ └──────┘ └───────┘ └───────┘
```

---

## 一、行业基础

### 1.1 在线广告发展史
- 1994 — 第一个 Banner 广告 (AT&T on HotWired)
- 搜索广告时代 (Google AdWords, 2000)
- 展示广告网络化 (Ad Network, 2003+)
- 程序化购买兴起 (RTB, 2009+)
- 移动广告爆发 (2012+)
- 信息流广告 & 短视频广告 (2016+)
- 隐私保护时代 (GDPR/CCPA/个保法, 2018+)

### 1.2 核心商业模式
- **CPM** (Cost Per Mille) — 千次展示计费
- **CPC** (Cost Per Click) — 点击计费
- **CPA** (Cost Per Action) — 转化计费
- **CPV** (Cost Per View) — 视频观看计费
- **OCPM/OCPC** — 优化目标出价
- **CPS** (Cost Per Sale) — 销售分成
- **CPL** (Cost Per Lead) — 线索计费

### 1.3 核心度量指标
- **曝光量 / 点击量 / 转化量**
- **CTR** (Click-Through Rate)
- **CVR** (Conversion Rate)
- **ECPM** (Effective CPM)
- **ROI / ROAS**
- **LTV** (Lifetime Value)
- **Fill Rate / Win Rate**

---

## 二、广告生态角色

### 2.1 广告主侧 (Demand Side)
- **广告主类型**: 品牌广告主 / 效果广告主 / 中小广告主
- **DSP** (Demand-Side Platform)
- **DMP** (Data Management Platform)
- **广告投放平台**: Google Ads / Meta Ads / 巨量引擎 / 磁力引擎 / 腾讯广告
- **归因与监测**: MMP (AppsFlyer, Adjust, 热云) / 第三方监测 (秒针, 尼尔森)

### 2.2 媒体侧 (Supply Side)
- **SSP** (Supply-Side Platform)
- **Ad Server** (广告服务器)
- **媒体类型**: 搜索引擎 / 社交媒体 / 视频平台 / 新闻资讯 / 工具类 App
- **广告位管理**: 瀑布流 (Waterfall) vs Header Bidding
- **流量变现策略**

### 2.3 中间平台
- **Ad Exchange** (广告交易平台)
- **RTB** (Real-Time Bidding) 协议
- **PMP** (Private Marketplace)
- **Preferred Deal / Programmatic Guaranteed**

---

## 三、广告系统架构

### 3.1 投放引擎 (Ad Serving)
- 广告请求处理流程
- 广告检索 (Ad Retrieval / Matching)
- 粗排 → 精排 → 重排
- 频次控制 (Frequency Capping)
- 预算控制 (Pacing / Budget Smoothing)
- 竞价机制 (GSP / VCG / First-Price / Second-Price)

### 3.2 定向系统 (Targeting)
- **人群定向**: 年龄/性别/地域/兴趣/行为
- **上下文定向**: 关键词/内容分类/语义
- **重定向** (Retargeting / Remarketing)
- **Lookalike 扩量**
- **设备/网络/时段定向**

### 3.3 创意系统
- 创意格式: 图片 / 视频 / 原生 / 互动 / 开屏
- 动态创意优化 (DCO)
- 创意审核 (自动化 + 人工)
- 素材 CDN 分发

### 3.4 计费与反作弊
- 计费点: 曝光计费 / 点击计费 / 转化计费
- 反作弊 (Anti-Fraud): 无效流量识别 (IVT)
- 归因窗口与归因模型
- 广告可见性 (Viewability)

---

## 四、数据与算法

### 4.1 用户画像 (User Profile)
- 用户标识: DeviceID / IDFA / OAID / Cookie / UnionID
- 标签体系: 基础属性 / 兴趣标签 / 行为标签 / 消费能力
- 画像构建: 实时特征 vs 离线特征
- ID Mapping / 跨设备识别

### 4.2 CTR/CVR 预估
- **经典模型**: LR → GBDT+LR → FM/FFM → DeepFM → DIN → DIEN
- **特征工程**: 用户特征 / 广告特征 / 上下文特征 / 交叉特征
- **样本工程**: 正负样本定义 / 样本加权 / 延迟转化
- **模型校准** (Calibration)
- **在线学习** (Online Learning)

### 4.3 出价与预算优化
- **eCPM 排序**: bid × pCTR × pCVR
- **智能出价**: oCPM / oCPC / tCPA / tROAS
- **预算分配**: 预算平滑 (Pacing) / 多计划预算分配
- **自动出价算法**: PID 控制 / 强化学习

### 4.4 推荐与广告融合
- 推荐系统与广告系统的异同
- 混排策略 (Ads-Organic Blending)
- 用户体验与商业化平衡
- 广告负反馈机制

### 4.5 搜索广告
- 查询理解 (Query Understanding)
- 广告相关性 (Relevance)
- 质量分 (Quality Score)
- 关键词匹配: 精确 / 短语 / 广泛

---

## 五、大数据基础设施

### 5.1 数据采集
- 客户端埋点: SDK / Pixel / S2S
- 日志采集: Flume / Filebeat / Logstash
- 数据格式: Protobuf / Avro / JSON

### 5.2 实时计算
- **消息队列**: Kafka / Pulsar / RocketMQ
- **流处理**: Flink / Spark Streaming / Storm
- **实时场景**: 实时特征 / 实时反作弊 / 实时报表 / 实时预算控制

### 5.3 离线计算
- **数据仓库**: Hive / Spark SQL / Presto / ClickHouse
- **数仓分层**: ODS → DWD → DWS → ADS
- **调度系统**: Airflow / DolphinScheduler / Azkaban
- **数据质量**: 数据校验 / 数据对账 / 数据血缘

### 5.4 数据存储
- **OLAP**: ClickHouse / Doris / Druid / StarRocks
- **KV 存储**: Redis / HBase / RocksDB
- **特征存储**: 在线特征服务 / 离线特征仓库
- **数据湖**: Hudi / Iceberg / Delta Lake

### 5.5 数据应用
- **报表系统**: 广告主报表 / 内部运营报表 / 实时大盘
- **数据分析**: 多维分析 / 归因分析 / 漏斗分析
- **AB 实验平台**: 分流 / 指标计算 / 显著性检验
- **数据服务**: API 网关 / 数据中台

---

## 六、工程实践

### 6.1 高性能广告系统
- 低延迟要求 (< 100ms 端到端)
- 高 QPS 处理 (百万级)
- 服务架构: 微服务 / Service Mesh
- 缓存策略: 多级缓存

### 6.2 模型工程
- 特征平台 (Feature Store)
- 模型训练: 参数服务器 / 分布式训练
- 模型部署: TensorFlow Serving / TorchServe / Triton
- 模型监控: 效果监控 / 漂移检测

### 6.3 稳定性保障
- 限流 / 熔断 / 降级
- 灰度发布 / 蓝绿部署
- 容灾与多机房
- 监控告警体系

---

## 七、行业合规与隐私

### 7.1 隐私法规
- **GDPR** (欧盟)
- **CCPA/CPRA** (加州)
- **个人信息保护法** (中国)
- **COPPA** (儿童隐私)

### 7.2 隐私技术
- **差分隐私** (Differential Privacy)
- **联邦学习** (Federated Learning)
- **隐私沙盒** (Privacy Sandbox / Topics API)
- **SKAdNetwork / SKAN** (iOS 归因)
- Cookie 消亡与替代方案

### 7.3 广告审核与合规
- 广告法与行业规范
- 内容审核 (文本/图片/视频)
- 行业准入 (金融/医疗/教育)

---

## 八、主要玩家与平台

### 8.1 海外
- **Google** (Search Ads / Display / YouTube / AdMob)
- **Meta** (Facebook / Instagram / Audience Network)
- **Amazon** (Sponsored Ads / DSP)
- **TikTok for Business**
- **Microsoft** (Bing Ads / LinkedIn)
- **The Trade Desk / DV360 / Criteo**

### 8.2 国内
- **字节跳动** (巨量引擎 / 穿山甲)
- **腾讯** (腾讯广告 / 优量汇)
- **阿里巴巴** (阿里妈妈 / Uni Marketing)
- **百度** (百度营销 / 百青藤)
- **快手** (磁力引擎)
- **美团 / 拼多多 / 小红书 / B站**

---

## 九、前沿趋势

- **AIGC 与广告创意** — AI 生成素材 / 智能文案
- **大模型在广告中的应用** — 意图理解 / 智能投放助手
- **RTA** (Real-Time API) — 广告主实时决策
- **sCTR / 深度转化优化** — 更深层次的转化目标
- **CTV / OTT 广告** — 联网电视广告
- **零售媒体网络** (Retail Media Network)
- **注意力经济** — 注意力度量替代传统指标

---

## 项目结构

```
awesome-ads/
├── README.md                  # 知识地图总览 (本文件)
├── 01-industry-basics/        # 行业基础
│   ├── history.md             # 发展史
│   ├── business-models.md     # 商业模式
│   └── metrics.md             # 核心指标
├── 02-ecosystem/              # 生态角色
│   ├── demand-side.md         # 广告主侧
│   ├── supply-side.md         # 媒体侧
│   └── exchange.md            # 交易平台
├── 03-ad-system/              # 广告系统架构
│   ├── serving.md             # 投放引擎
│   ├── targeting.md           # 定向系统
│   ├── creative.md            # 创意系统
│   └── billing-antifraud.md   # 计费与反作弊
├── 04-data-algorithm/         # 数据与算法
│   ├── user-profile.md        # 用户画像
│   ├── ctr-cvr.md             # CTR/CVR 预估
│   ├── bidding.md             # 出价优化
│   ├── rec-ads-blend.md       # 推荐广告融合
│   └── search-ads.md          # 搜索广告
├── 05-big-data-infra/         # 大数据基础设施
│   ├── collection.md          # 数据采集
│   ├── realtime.md            # 实时计算
│   ├── offline.md             # 离线计算
│   ├── storage.md             # 数据存储
│   └── application.md         # 数据应用
├── 06-engineering/            # 工程实践
│   ├── high-performance.md    # 高性能系统
│   ├── ml-engineering.md      # 模型工程
│   └── reliability.md         # 稳定性保障
├── 07-compliance/             # 合规与隐私
│   ├── regulations.md         # 隐私法规
│   ├── privacy-tech.md        # 隐私技术
│   └── ad-review.md           # 广告审核
├── 08-players/                # 主要玩家
│   ├── overseas.md            # 海外平台
│   └── china.md               # 国内平台
├── 09-trends/                 # 前沿趋势
│   └── trends.md
└── resources/                 # 学习资源
    ├── papers.md              # 经典论文
    ├── books.md               # 推荐书籍
    └── talks.md               # 演讲与课程
```

---

## 整理思路与协作方式

### 我们的整理原则

1. **自顶向下** — 先搭骨架，再填内容，逐步细化
2. **实战导向** — 优先覆盖工作中高频使用的知识点
3. **大数据视角** — 你是大数据开发，重点深挖数据侧和工程侧
4. **中国市场为主** — 兼顾海外，但以国内广告生态为主线

### 推荐整理顺序

| 阶段 | 内容 | 原因 |
|------|------|------|
| Phase 1 | 行业基础 + 生态角色 | 建立全局认知 |
| Phase 2 | 大数据基础设施 | 你的核心技能领域 |
| Phase 3 | 广告系统架构 | 理解业务系统全貌 |
| Phase 4 | 数据与算法 | 理解算法团队的工作 |
| Phase 5 | 工程实践 | 提升系统设计能力 |
| Phase 6 | 合规/玩家/趋势 | 拓展行业视野 |

### 每个主题的标准模板

```markdown
# 主题名称

## 一句话概述
## 核心概念
## 架构/流程图
## 关键技术点
## 与大数据开发的关联
## 面试高频问题
## 推荐阅读
```

---

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
