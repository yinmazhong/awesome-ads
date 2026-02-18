# 面试 Q&A Part 3: 大数据基础设施 + 工程 + 合规 + 玩家

## 五、大数据基础设施 (Big Data Infra)

### 5.1 数据采集 (Collection)

#### Q69: 广告数据采集整体架构？ / Ad data collection architecture?

客户端 (SDK 埋点) + 服务端 (日志) → 采集层 (Nginx/Flume) → 传输层 (Kafka) → 消费层 (Flink 实时 / Spark 离线) → 存储层 (HDFS/HBase/ClickHouse)。

Client SDK + Server logs → Collection (Nginx/Flume) → Transport (Kafka) → Consumption (Flink realtime / Spark offline) → Storage (HDFS/HBase/ClickHouse).

---

#### Q70: 客户端埋点 vs 服务端日志？ / Client tracking vs server logging?

| 维度 | 客户端 Client | 服务端 Server |
|------|-------------|-------------|
| 数据 | 用户行为（曝光/点击） | 系统事件（请求/竞价） |
| 可靠性 | 受网络影响，可能丢失 | 高可靠 |
| 实时性 | 有延迟（批量上报） | 实时 |
| 防作弊 | 易被篡改 | 难篡改 |

Best practice: 两者结合，客户端采集用户行为，服务端记录系统事件，交叉验证。

---

#### Q71: 如何保证数据采集不丢数据？ / How to ensure no data loss?

1. **Kafka 多副本** + acks=all
2. 客户端**本地缓存 + 重试**
3. 端到端 **Exactly-Once**（Kafka 事务）
4. **数据对账**（采集量 vs 消费量）
5. **监控告警**（延迟/丢失/重复阈值）

---

#### Q72: Kafka 在广告数据采集中的作用？ / Kafka's role in ad data collection?

Kafka 作为**消息中间件**，解耦数据生产和消费：

- **缓冲**：应对流量峰值（如大促期间）
- **解耦**：生产者（埋点/日志）和消费者（Flink/Spark）独立扩展
- **持久化**：消息持久化到磁盘，防止数据丢失
- **多消费者**：同一份数据被实时计费、实时特征、离线分析等多个系统消费

配置要点：分区数（并行度）、副本数（可靠性）、retention（保留时间）。

---

### 5.2 实时计算 (Realtime)

#### Q73: Flink vs Spark Streaming？为什么广告选 Flink？ / Why Flink for ad tech?

| 维度 | Flink | Spark Streaming |
|------|-------|----------------|
| 模型 | 真正流处理 True streaming | 微批 Micro-batch |
| 延迟 | 毫秒级 ms | 秒级 seconds |
| 状态管理 | 原生支持 Native | 需外部存储 |
| Exactly-Once | 原生 Native | 需额外配置 |

广告选 Flink：实时预算控制需毫秒级延迟；实时特征需状态管理；反作弊需流式检测。

---

#### Q74: Flink Exactly-Once 如何实现？ / How does Flink implement Exactly-Once?

**Checkpoint + 两阶段提交 Two-Phase Commit**：
1. Checkpoint Barrier 对齐所有算子状态
2. 状态快照持久化到分布式存储
3. Sink 端两阶段提交（预提交 → Checkpoint 完成 → 正式提交）
4. 故障恢复时从最近 Checkpoint 恢复

---

#### Q75: Flink 状态管理和 Checkpoint？ / Flink state management and checkpointing?

**状态类型**：ValueState、ListState、MapState、ReducingState。

**Checkpoint 机制**：JobManager 定期触发 → Barrier 注入 Source → Barrier 流经所有算子 → 每个算子异步快照状态到 StateBackend (RocksDB/HDFS) → 全部完成后 Checkpoint 成功。

**配置要点**：Checkpoint 间隔（通常 1-5 min）、StateBackend 选择（RocksDB 适合大状态）、增量 Checkpoint。

---

#### Q76: 如何处理 Flink 数据倾斜？ / How to handle Flink data skew?

1. **Key 打散**：热点 Key 加随机后缀，先局部聚合再全局聚合
2. **Rebalance**：`rebalance()` 均匀分发
3. **Local-Global 聚合**：两阶段聚合
4. **调整并行度**：增加热点算子并行度
5. **过滤热点**：极端热点单独处理

---

#### Q77: 实时预算控制的实现方案？ / Real-time budget control implementation?

Flink 消费 Kafka 计费日志 → 实时聚合每个广告计划的消耗 → 写入 Redis（预算余额）→ 投放引擎每次请求查询 Redis 判断预算是否充足。

关键：**最终一致性**（允许短暂超支）+ **预扣机制**（竞价成功时预扣，展示确认后正式扣费）+ **兜底检查**（定期与数据库对账）。

---

### 5.3 离线计算 (Offline)

#### Q78: 数据仓库分层架构各层职责？ / Data warehouse layer responsibilities?

| 层 Layer | 职责 Role | 广告示例 Ad Example |
|---------|----------|-------------------|
| **ODS** | 原始数据，贴源 Raw | 原始曝光/点击日志 |
| **DWD** | 明细数据，清洗标准化 Cleaned | 清洗后事件明细表 |
| **DWS** | 汇总数据，轻度聚合 Aggregated | 广告计划维度日汇总 |
| **ADS** | 应用数据，面向业务 Application | 广告主报表、运营看板 |

---

#### Q79: 广告数仓核心表？如何设计？ / Core ad data warehouse tables?

- **事实表**：曝光事实表、点击事实表、转化事实表、计费事实表
- **维度表**：广告主维度、广告计划维度、创意维度、广告位维度、时间维度

设计原则：星型模型 (Star Schema)，事实表存度量值（曝光数/点击数/花费），维度表存描述信息。粒度：通常到广告计划 × 广告位 × 小时。

---

#### Q80: Hive vs Spark？ / Hive vs Spark?

| | Hive | Spark |
|--|------|-------|
| 引擎 | MapReduce/Tez | 内存计算 |
| 速度 | 慢 | 快 10-100x |
| 适用 | 超大规模批处理 | 迭代计算/ML |

实际：Spark 已成主流，Hive 主要作元数据管理 (Metastore)。Spark is now mainstream; Hive mainly serves as metadata management.

---

#### Q81: 如何保证离线数据质量？ / How to ensure offline data quality?

1. **Schema 校验**：字段类型、非空约束
2. **数据量监控**：与前一天对比，波动超阈值告警
3. **业务规则校验**：如 CTR 不应超过 100%
4. **跨表一致性**：曝光数 ≥ 点击数 ≥ 转化数
5. **数据血缘**：追踪数据来源和加工链路
6. **DQC (Data Quality Check)**：每个 ETL 任务后自动检查

---

#### Q82: 数据倾斜如何处理？ / How to handle data skew?

1. 热点 Key **加盐**（随机后缀）→ 两阶段聚合
2. Map 端**预聚合** (Combiner)
3. **广播小表**（避免 Shuffle）
4. 调整**分区数**
5. 采样分析倾斜 Key，**单独处理**

---

#### Q83: 调度系统如何保证任务依赖和重试？ / Task scheduling: dependencies and retries?

调度系统（Airflow/DolphinScheduler）：
- **DAG 依赖**：定义任务间依赖关系，上游完成才触发下游
- **重试机制**：失败自动重试 N 次，间隔递增
- **告警**：失败/超时通知（邮件/钉钉/飞书）
- **回填 (Backfill)**：支持历史数据重跑
- **SLA 监控**：关键任务设置完成时间 SLA

---

### 5.4 存储 (Storage)

#### Q84: ClickHouse vs Doris？ / ClickHouse vs Doris?

| 维度 | ClickHouse | Doris |
|------|-----------|-------|
| 架构 | Shared-Nothing | MPP |
| 并发 | 低（大查询） | 高（多用户） |
| 更新 | 不擅长 | 支持 Upsert |
| 运维 | 复杂 | 简单 |
| 适用 | 内部分析 | **面向用户报表** |

ClickHouse excels at large analytical queries; Doris is better for high-concurrency user-facing reports.

---

#### Q85: Redis 在广告系统中的应用？ / Redis use cases in ad systems?

1. **频次控制**：用户看过广告的次数
2. **预算缓存**：实时预算余额
3. **特征缓存**：用户/广告实时特征
4. **人群包**：Bitmap 存储人群标签
5. **限流**：接口限流计数器
6. **分布式锁**：预算扣减并发控制

---

#### Q86: Feature Store 架构？如何保证特征一致性？ / Feature Store and consistency?

架构：离线特征 (Spark→HBase/Hive) + 实时特征 (Flink→Redis) → 统一 Feature Store API → 训练/推理。

一致性：**同一份特征定义** (DSL) 生成离线和在线特征 + 定期对比差异 + 特征版本管理。

Same feature definition (DSL) generates both offline and online features; periodic consistency checks; feature versioning.

---

#### Q87: 数据湖 (Hudi/Iceberg) 解决什么问题？ / What problems do data lakes solve?

1. **ACID 事务**：支持更新/删除（传统 HDFS 不支持）
2. **增量读取**：只读取变更数据，提升效率
3. **Schema 演进**：支持字段增删改
4. **时间旅行**：查询历史版本数据
5. **流批一体**：同一份数据支持流式和批式读取

广告场景：用户画像增量更新、归因数据修正、GDPR 数据删除。

---

#### Q88: HBase RowKey 如何设计？ / HBase RowKey design?

原则：**避免热点** + **满足查询模式**。

广告场景示例：
- 用户特征表：`reverse(user_id)` — 反转避免热点
- 广告报表：`advertiser_id + date + campaign_id` — 支持按广告主+日期范围查询
- 实时计费：`hash(campaign_id) + timestamp` — 散列+时间有序

---

### 5.5 数据应用 (Application)

#### Q89: 广告报表系统技术架构？ / Ad reporting system architecture?

数据源 (Kafka/Hive) → ETL (Spark/Flink) → OLAP 引擎 (ClickHouse/Doris) → 查询服务 (API) → 前端 (BI/自研看板)。

关键：**预聚合+实时聚合**结合、缓存热点查询、分层存储（热数据 ClickHouse、冷数据 Hive）。

---

#### Q90: A/B 实验平台核心组件？分层实验？ / A/B testing platform and layered experiments?

**组件**：流量分配 (Hash 分桶)、实验管理 (创建/配置)、指标计算 (离线+实时)、统计检验 (t-test/卡方)。

**分层实验**：流量分为正交多层（UI 层/算法层/策略层），每层独立随机分桶，不同层实验互不干扰，最大化并行度。

Layered experiments: Traffic split into orthogonal layers (UI/algorithm/strategy), each independently randomized, enabling maximum parallel experimentation.

---

#### Q91: 如何判断实验结果是否显著？ / How to determine statistical significance?

1. **假设检验**：H0 (无差异) vs H1 (有差异)
2. **p-value < 0.05**：拒绝 H0，认为差异显著
3. **置信区间**：效果的 95% 置信区间不包含 0
4. **样本量**：需足够大（通常需数万~数十万样本）
5. **实验周期**：至少 7 天（覆盖周末效应）
6. **多重检验校正**：多个指标同时检验时用 Bonferroni 校正

---

## 六、工程 (Engineering)

### 6.1 高性能

#### Q92: 多级缓存设计和一致性？ / Multi-level cache design and consistency?

L1 (本地 Caffeine, ~1ms) → L2 (Redis 集群, ~3ms) → L3 (数据库, ~10ms)。

一致性：**Cache-Aside** 模式（先更新 DB，再删缓存）+ TTL 兜底 + 异步刷新 + 版本号校验。广告场景容忍短暂不一致（秒级）。

---

#### Q93: 模型推理如何优化延迟？ / How to optimize inference latency?

1. 模型**蒸馏** (大→小模型)
2. **量化** (FP32→INT8)
3. **TensorRT**/ONNX Runtime 加速
4. 算子融合
5. Batch 推理
6. 预计算 Embedding
7. GPU/专用硬件

---

### 6.2 ML 工程

#### Q94: 如何保证训练-推理特征一致性？ / Training-serving feature consistency?

1. **统一特征定义**：同一份 DSL 生成训练和在线特征
2. **Feature Store**：统一存储和服务
3. **特征日志**：在线推理时记录特征值，训练时回放
4. **一致性校验**：定期对比分布
5. **版本管理**：模型绑定特征版本

---

#### Q95: 广告模型分布式训练？参数服务器 vs 数据并行？ / Distributed training: PS vs Data Parallel?

| 方案 | 适用 | 特点 |
|------|------|------|
| **参数服务器 PS** | 稀疏模型 (大 Embedding) | PS 存参数，Worker 拉取/推送 |
| **数据并行 DP** | 稠密模型 (DNN) | 数据分片，AllReduce 同步 |

广告模型：Embedding 表 TB 级用 PS；DNN 部分用 DP。混合架构。

---

#### Q96: AUC vs GAUC？为什么广告用 GAUC？ / AUC vs GAUC?

AUC 衡量全局排序。但不同用户 CTR 基准不同（活跃用户天然高）。

**GAUC** = 按用户分组计算 AUC 再加权平均。衡量"**每个用户内部的排序能力**"，更贴合广告场景（给每个用户选最好的广告）。

GAUC = weighted average of per-user AUCs. Measures ranking quality within each user, which is what ad serving actually needs.

---

#### Q97: 如何监控线上模型效果？数据漂移检测？ / Online model monitoring and drift detection?

1. **效果指标**：实时监控 CTR/CVR/AUC/GAUC，与基线对比
2. **校准度**：监控预估值 vs 实际值的偏差
3. **特征漂移**：监控特征分布变化 (PSI/KL 散度)
4. **标签漂移**：监控转化率基准变化
5. **告警**：指标异常自动告警 + 自动回滚

---

### 6.3 可靠性

#### Q98: 限流、熔断、降级？ / Rate limiting, circuit breaking, degradation?

- **限流 Rate Limiting**：控制请求速率防过载。算法：令牌桶、滑动窗口。
- **熔断 Circuit Breaking**：下游异常时快速失败防级联。状态：Closed→Open→Half-Open。
- **降级 Degradation**：过载时关闭非核心功能。如：关精排用粗排、返回默认广告。

---

#### Q99: 灰度发布流程？ / Canary release process?

新版本部署少量机器 (1%) → 监控核心指标（延迟/错误率/CTR/收入）→ 无异常逐步扩大 (5%→20%→50%→100%) → 有异常立即回滚 → 全量后观察 24h。

---

#### Q100: 如何做到 99.99% 可用性？ / How to achieve 99.99% availability?

99.99% = 全年停机 <52 分钟。

多机房部署（主备/双活）+ 自动故障转移 + 限流熔断降级 + 灰度发布 + 全链路监控 + 故障演练 (Chaos Engineering) + SLA 分级。

---

#### Q101: 多机房容灾方案？ / Multi-datacenter disaster recovery?

| 方案 | 说明 | 复杂度 |
|------|------|--------|
| 冷备 Cold Standby | 备机房不接流量，故障时切换 | 低 |
| 热备 Hot Standby | 备机房接少量流量，随时接管 | 中 |
| 双活 Active-Active | 两机房同时接流量 | 高 |

广告系统通常：核心链路双活，数据层主从复制，故障自动切换 <30s。

---

## 七、合规 (Compliance)

### 7.1 隐私技术

#### Q102: 联邦学习原理？广告应用？ / Federated learning in advertising?

原理：多方不共享原始数据，协作训练模型。各方本地训练 → 上传梯度/参数 → 聚合服务器聚合 → 下发更新参数。数据不出域。

广告应用：广告平台+广告主联合建模（纵向联邦）；多平台联合训练（横向联邦）。提升效果同时保护隐私。

---

#### Q103: 差分隐私？ε 参数？ / Differential privacy and ε?

在数据中添加噪声，使单个用户是否参与不显著影响输出。

**ε (epsilon)** = 隐私预算。ε 越小 → 噪声越大 → 隐私越强 → 可用性越低。典型 ε = 1-10。

---

#### Q104: Google Privacy Sandbox 核心 API？ / Privacy Sandbox APIs?

- **Topics API**：浏览器端推断兴趣主题（替代第三方 Cookie）
- **Protected Audience API (FLEDGE)**：设备端竞价（替代 Retargeting）
- **Attribution Reporting API**：隐私归因，聚合报告

---

#### Q105: SKAdNetwork 工作原理和限制？ / SKAdNetwork mechanism and limitations?

原理：Apple 作为可信中间方，在设备端完成归因，只向广告网络发送聚合的、延迟的转化通知。

限制：24-48h 延迟、仅 64 转化值、无用户级数据、单一归因来源、无法实时优化。

---

#### Q106: 数据清洁室解决什么问题？ / What does Data Clean Room solve?

多方在**安全环境**中联合分析数据（用户匹配/归因分析），只输出**聚合结果**，不暴露个人数据。

代表：Google Ads Data Hub、巨量云图、AWS Clean Rooms、Meta Advanced Analytics。

---

### 7.2 广告审核

#### Q107: 广告法核心条款？ / Key advertising law provisions?

- 禁止虚假广告、误导性宣传
- 禁止"最"、"第一"等绝对化用语
- 特殊行业需资质（医疗/金融/教育）
- 广告需可识别（标注"广告"）
- 保护未成年人
- 个人信息保护（需同意才能定向）

---

### 7.3 隐私法规

#### Q108: GDPR 对广告的主要影响？ / GDPR impact on advertising?

1. 处理个人数据需**合法基础**（同意/合法利益）
2. 用户权利：访问/删除/可携带
3. **数据最小化**
4. 需明确**同意** (CMP 弹窗)
5. 罚款最高 2000 万欧元或全球年营收 4%
6. 第三方数据使用受限

---

#### Q109: 中国个保法对广告定向的要求？ / China PIPL requirements for ad targeting?

- 处理个人信息需**同意**
- 个性化推荐需提供**关闭选项**
- 禁止**大数据杀熟**
- 敏感信息需**单独同意**
- 跨境传输需**安全评估**

---

#### Q110: 第三方 Cookie 淘汰后如何应对？ / Response to 3P cookie deprecation?

1. Google Topics API
2. UID 2.0（基于邮箱加密标识）
3. **第一方数据** (CRM/CDP)
4. **上下文定向**复兴
5. **数据清洁室**
6. 服务端追踪 (Conversions API)

---

## 八、国内玩家 (China Players)

#### Q111: 国内广告市场主要玩家和核心优势？ / Major China ad players?

| 玩家 Player | 份额 Share | 核心优势 Core Advantage |
|------------|-----------|----------------------|
| 字节跳动 ByteDance | ~28% | 推荐算法、短视频、穿山甲联盟 |
| 阿里巴巴 Alibaba | ~20% | 电商数据、交易闭环、ROAS |
| 腾讯 Tencent | ~16% | 微信生态、社交数据、小程序 |
| 百度 Baidu | ~7% | 搜索意图、oCPC 先驱 |
| 快手 Kuaishou | ~5% | 下沉市场、直播电商 |

---

#### Q112: 什么是围墙花园？ / What is a Walled Garden?

国内各大平台自建广告闭环，数据不互通。广告主 → 巨量引擎 → 抖音/头条（数据不出字节）；广告主 → 腾讯广告 → 微信/QQ（数据不出腾讯）。

影响：独立 DSP 空间被压缩、需多平台分别投放、跨平台归因困难、数据孤岛严重。

Each major platform operates a closed-loop ad ecosystem where data doesn't leave the platform. This compresses independent DSP space, requires separate campaign management per platform, and makes cross-platform attribution difficult.

---

#### Q113: 什么是 RTA？为什么国内需要？ / What is RTA and why is it needed?

RTA (Real-Time API)：媒体向广告主 RTA 服务发实时请求 (<50ms)，广告主返回是否参竞/出价调整/人群标签。

**为什么需要**：围墙花园下广告主无法在媒体平台上使用自有数据。RTA 让广告主将自有数据能力实时输出到媒体广告决策中。

RTA compensates for data isolation in walled gardens by letting advertisers inject their own data signals into the media's real-time ad decision process.

---

#### Q114: 电商广告 vs 信息流广告？ / E-commerce ads vs in-feed ads?

| | 电商广告 (阿里/拼多多) | 信息流广告 (字节/腾讯) |
|--|---------------------|---------------------|
| 场景 | 购物场景，强购买意图 | 内容消费，弱购买意图 |
| 核心指标 | ROAS, GMV | CPA, 转化量 |
| 定向 | 商品/品类/搜索词 | 兴趣/行为/人群 |
| 闭环 | 广告→购买完整闭环 | 需跳转外部落地页 |
| 计费 | CPC (直通车) / oCPM | oCPM 为主 |
