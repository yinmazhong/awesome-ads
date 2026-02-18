# 实时计算 (Real-Time Computing)

## 一句话概述

实时计算是广告系统的"神经系统"，通过 Kafka + Flink 等技术栈实现毫秒到秒级的数据处理，支撑实时特征、实时反作弊、实时预算控制、实时报表等核心场景。

---

## 实时计算在广告系统中的位置

```
数据源                  实时计算层                    应用层
┌──────────┐        ┌──────────────┐          ┌──────────────┐
│ 曝光日志  │──┐     │              │     ┌───►│ 实时特征服务  │
│ 点击日志  │──┤     │              │     │    └──────────────┘
│ 转化日志  │──┼────►│  Flink 集群  │─────┤    ┌──────────────┐
│ 竞价日志  │──┤     │              │     ├───►│ 实时反作弊    │
│ 行为日志  │──┘     │              │     │    └──────────────┘
└──────────┘        └──────────────┘     │    ┌──────────────┐
      │                                   ├───►│ 实时预算控制  │
      ▼                                   │    └──────────────┘
┌──────────┐                              │    ┌──────────────┐
│  Kafka   │                              └───►│ 实时报表/大盘 │
└──────────┘                                   └──────────────┘
```

---

## 消息队列

### Kafka

| 特性 | 说明 |
|------|------|
| **吞吐量** | 单机百万级 TPS |
| **延迟** | 毫秒级 |
| **持久化** | 磁盘顺序写，可保留数天 |
| **消费模式** | Consumer Group，支持多消费者 |
| **分区** | 水平扩展，并行消费 |

**广告系统中的 Kafka 使用**:

```
生产者:
  - 广告服务器写入请求/展示/点击日志
  - 数据接收服务写入客户端上报数据
  - 转化回传服务写入转化数据

消费者:
  - Flink 实时计算任务
  - 实时特征更新服务
  - 实时监控服务
  - 数据入湖/入仓任务 (Kafka → HDFS/Hive)
```

### Kafka vs Pulsar vs RocketMQ

| 维度 | Kafka | Pulsar | RocketMQ |
|------|-------|--------|----------|
| **吞吐** | 极高 | 高 | 高 |
| **延迟** | 毫秒 | 毫秒 | 毫秒 |
| **存储** | Broker 本地 | BookKeeper 分离 | Broker 本地 |
| **多租户** | 弱 | 强 | 中 |
| **国内使用** | 广泛 | 增长中 | 阿里系广泛 |
| **适用场景** | 大数据管道 | 多租户/云原生 | 业务消息 |

---

## Flink 实时计算

### 为什么选 Flink？

| 特性 | Flink | Spark Streaming | Storm |
|------|-------|-----------------|-------|
| **处理模型** | 真正流处理 | 微批 (Micro-batch) | 流处理 |
| **延迟** | 毫秒级 | 秒级 | 毫秒级 |
| **Exactly-Once** | 支持 | 支持 | At-least-once |
| **窗口** | 丰富 (滚动/滑动/会话) | 基础 | 基础 |
| **状态管理** | 强 (RocksDB State) | 弱 | 弱 |
| **SQL 支持** | Flink SQL | Spark SQL | 无 |
| **生态** | 丰富 | 最丰富 | 一般 |

### Flink 在广告中的核心场景

---

## 场景一: 实时特征计算

```
需求: 计算用户最近 1 小时的行为特征

Flink SQL:
SELECT
  user_id,
  COUNT(*) as click_cnt_1h,
  COUNT(DISTINCT ad_id) as click_ad_cnt_1h,
  SUM(CASE WHEN category = 'game' THEN 1 ELSE 0 END) as game_click_1h
FROM ad_click_stream
WHERE event_time >= NOW() - INTERVAL '1' HOUR
GROUP BY
  user_id,
  HOP(event_time, INTERVAL '1' MINUTE, INTERVAL '1' HOUR)

输出: 写入 Redis，供在线特征服务查询
```

```java
// Flink DataStream API 示例
DataStream<ClickEvent> clicks = env
    .addSource(new FlinkKafkaConsumer<>("ads.click", schema, props));

clicks
    .keyBy(event -> event.getUserId())
    .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(1)))
    .aggregate(new ClickFeatureAggregator())
    .addSink(new RedisSink<>(redisConfig, new FeatureRedisMapper()));
```

---

## 场景二: 实时反作弊

```
需求: 实时检测异常点击行为

规则示例:
  - 同一 IP 1分钟内点击 > 10 次 → 标记异常
  - 同一设备 5分钟内点击不同广告 > 20 个 → 标记异常
  - 点击到转化时间 < 1 秒 → 标记异常

Flink 实现:
clicks
    .keyBy(event -> event.getIp())
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new CountAggregator())
    .filter(count -> count.getValue() > 10)
    .addSink(new FraudAlertSink());

CEP (Complex Event Processing):
Pattern<ClickEvent, ?> pattern = Pattern
    .<ClickEvent>begin("first").where(isClick())
    .followedBy("rapid").where(isClick())
    .within(Time.seconds(1))
    .times(10);

CEP.pattern(clicks.keyBy("deviceId"), pattern)
    .select(new FraudPatternSelector());
```

---

## 场景三: 实时预算控制

```
需求: 实时计算广告计划的消耗金额，防止超预算

流程:
  展示/点击事件 → Flink 计算消耗 → 更新 Redis 预算余额

Flink 实现:
billingEvents
    .keyBy(event -> event.getCampaignId())
    .process(new BudgetControlFunction())  // 有状态处理
    .addSink(new RedisBudgetSink());

// BudgetControlFunction 内部:
// 1. 累加消耗金额
// 2. 检查是否超预算
// 3. 超预算 → 发送暂停信号到广告投放引擎
// 4. 更新 Redis 中的剩余预算

关键要求:
  - 延迟: < 1秒 (避免超预算)
  - 精确性: Exactly-Once 语义
  - 高可用: 故障恢复后状态不丢失
```

---

## 场景四: 实时报表

```
需求: 实时计算广告效果指标 (展示量/点击量/消耗/CTR)

Flink SQL:
INSERT INTO realtime_report
SELECT
  campaign_id,
  TUMBLE_START(event_time, INTERVAL '1' MINUTE) as stat_time,
  COUNT(CASE WHEN event_type = 'impression' THEN 1 END) as impressions,
  COUNT(CASE WHEN event_type = 'click' THEN 1 END) as clicks,
  SUM(CASE WHEN event_type = 'impression' THEN cost ELSE 0 END) as cost,
  CAST(clicks AS DOUBLE) / NULLIF(impressions, 0) as ctr
FROM ad_event_stream
GROUP BY
  campaign_id,
  TUMBLE(event_time, INTERVAL '1' MINUTE)

输出: 写入 ClickHouse / Doris，供报表系统查询
```

---

## 场景五: 实时样本拼接

```
需求: 将曝光、点击、转化事件拼接成训练样本

流程:
  曝光事件 (T=0)
    ↓ 等待点击 (窗口: 30分钟)
  点击事件 (T=5s)
    ↓ 等待转化 (窗口: 24小时)
  转化事件 (T=2h)
    ↓
  完整样本: {曝光特征, 点击标签=1, 转化标签=1}

Flink 实现:
  - 使用 Interval Join 或 CoGroup
  - 曝光流 JOIN 点击流 (30分钟窗口)
  - 点击流 JOIN 转化流 (24小时窗口)
  - 未匹配的曝光 → 负样本 (未点击)
  - 未匹配的点击 → 负样本 (未转化)

挑战:
  - 转化延迟长 (可能数天)
  - 状态量大 (需要保存所有未匹配的曝光)
  - 需要 RocksDB State Backend
```

---

## Flink 状态管理

### State Backend

| Backend | 存储 | 容量 | 速度 | 适用场景 |
|---------|------|------|------|---------|
| **MemoryState** | JVM Heap | 小 | 最快 | 小状态 |
| **FsState** | JVM Heap + 文件系统 | 中 | 快 | 中等状态 |
| **RocksDBState** | RocksDB (磁盘) | 大 | 中 | 大状态 (推荐) |

### Checkpoint 与容错

```
Checkpoint 配置:
  env.enableCheckpointing(60000);  // 每60秒
  env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
  env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
  env.setStateBackend(new RocksDBStateBackend("hdfs:///flink/checkpoints"));

容错流程:
  1. 定期 Checkpoint: 将状态快照到 HDFS
  2. 任务失败: 从最近的 Checkpoint 恢复
  3. 重放 Kafka 数据: 从 Checkpoint 对应的 offset 开始
  4. 恢复到一致状态: Exactly-Once 语义
```

---

## Flink 性能优化

### 常见优化手段

| 优化项 | 说明 |
|--------|------|
| **并行度调优** | 根据数据量和 Kafka 分区数设置合理并行度 |
| **State TTL** | 设置状态过期时间，避免状态无限增长 |
| **增量 Checkpoint** | RocksDB 增量 Checkpoint，减少 CP 时间 |
| **异步 IO** | 外部查询使用 AsyncFunction |
| **Mini-batch** | Flink SQL 开启 mini-batch 减少状态访问 |
| **数据倾斜** | 两阶段聚合 (Local + Global) |

### 数据倾斜处理

```
问题: 某些 key (如热门广告主) 数据量远大于其他 key

方案: 两阶段聚合

// 第一阶段: 加随机前缀，分散到多个 subtask
stream
    .map(event -> {
        event.setKey(event.getKey() + "_" + random.nextInt(10));
        return event;
    })
    .keyBy("key")
    .window(...)
    .aggregate(new LocalAggregator())  // 局部聚合

// 第二阶段: 去掉前缀，全局聚合
    .map(result -> {
        result.setKey(result.getKey().split("_")[0]);
        return result;
    })
    .keyBy("key")
    .window(...)
    .aggregate(new GlobalAggregator())  // 全局聚合
```

---

## 实时数据质量保障

### 数据延迟监控

```
监控指标:
  - Kafka Consumer Lag: 消费延迟的消息数
  - Event Time Lag: 事件时间与处理时间的差值
  - End-to-End Latency: 从数据产生到结果输出的总延迟

告警规则:
  Consumer Lag > 100,000 → P2 告警
  Consumer Lag > 1,000,000 → P1 告警
  E2E Latency > 5min → P1 告警
```

### 数据完整性

```
实时 vs 离线对账:
  实时计算结果 (Flink) vs 离线计算结果 (Hive/Spark)
  
  差异率 = |实时值 - 离线值| / 离线值
  
  可接受差异: < 1% (由于时间窗口边界等原因)
  需要排查: > 5%
```

---

## 与大数据开发的日常工作

- **Flink 任务开发**: 编写实时计算逻辑 (DataStream / Flink SQL)
- **Kafka 集群运维**: Topic 管理、分区调整、性能调优
- **状态管理**: Checkpoint 配置、状态清理、故障恢复
- **性能调优**: 并行度、数据倾斜、反压处理
- **监控告警**: 延迟监控、数据质量监控
- **实时离线一致性**: 实时和离线计算结果的对账

---

## 面试高频问题

1. Flink 和 Spark Streaming 的区别？为什么广告系统选 Flink？
2. Flink 的 Exactly-Once 语义是如何实现的？
3. Flink 状态管理和 Checkpoint 机制？
4. 实时特征计算的架构是怎样的？
5. 如何处理 Flink 任务的数据倾斜？
6. 实时预算控制的实现方案？

---

## 推荐阅读

- 《Flink 实战与性能优化》
- 《Kafka 权威指南》
- [Apache Flink 官方文档](https://flink.apache.org/)
- [字节跳动 Flink 实践](https://tech.bytedance.com/)
