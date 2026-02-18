# 离线计算 (Offline Computing)

## 一句话概述

离线计算是广告大数据系统的"大脑"，通过 Hive/Spark 等批处理引擎对海量历史数据进行 ETL、特征计算、报表聚合和模型训练样本生成，是数据仓库和数据分析的核心。

---

## 离线计算架构

```
数据源                    计算引擎                    输出
┌──────────┐          ┌──────────────┐          ┌──────────────┐
│ Kafka    │──ETL────►│ Hive / Spark │─────────►│ 数据仓库     │
│ (实时日志)│          │              │          │ (Hive 表)    │
└──────────┘          │              │          └──────────────┘
┌──────────┐          │              │          ┌──────────────┐
│ HDFS     │─────────►│              │─────────►│ OLAP 引擎    │
│ (历史数据)│          │              │          │(ClickHouse)  │
└──────────┘          │              │          └──────────────┘
┌──────────┐          │              │          ┌──────────────┐
│ MySQL    │─────────►│              │─────────►│ 特征存储     │
│ (业务数据)│          └──────────────┘          │ (HBase/Redis)│
└──────────┘                │                   └──────────────┘
                            │
                     ┌──────┴──────┐
                     │  调度系统    │
                     │(Airflow/DS) │
                     └─────────────┘
```

---

## 数据仓库分层 (重点)

### 分层架构

```
┌─────────────────────────────────────────────────┐
│  ADS (Application Data Store) — 应用数据层       │
│  面向应用的聚合数据 (报表、API、导出)             │
├─────────────────────────────────────────────────┤
│  DWS (Data Warehouse Summary) — 汇总数据层       │
│  轻度聚合，按主题域汇总 (日/周/月粒度)            │
├─────────────────────────────────────────────────┤
│  DWD (Data Warehouse Detail) — 明细数据层         │
│  清洗后的明细数据，统一格式和口径                  │
├─────────────────────────────────────────────────┤
│  ODS (Operational Data Store) — 原始数据层        │
│  原始日志数据，保持原貌                           │
├─────────────────────────────────────────────────┤
│  DIM (Dimension) — 维度层                        │
│  维度表 (广告主、广告计划、广告位、地域等)          │
└─────────────────────────────────────────────────┘
```

### 广告数仓各层示例

#### ODS 层 (原始数据)

```sql
-- ODS: 原始广告请求日志
CREATE TABLE ods_ad_request_log (
  request_id    STRING,
  timestamp     BIGINT,
  user_id       STRING,
  device_id     STRING,
  ip            STRING,
  ad_slot_id    STRING,
  os            STRING,
  app_id        STRING,
  geo_city      STRING,
  raw_json      STRING    -- 原始 JSON 保留
)
PARTITIONED BY (dt STRING, hour STRING)
STORED AS ORC;

-- 数据来源: Kafka → Flink/Spark → HDFS → Hive 外表
```

#### DWD 层 (明细数据)

```sql
-- DWD: 广告展示明细表
CREATE TABLE dwd_ad_impression (
  request_id      STRING,
  event_time      TIMESTAMP,
  user_id         STRING,
  device_id       STRING,
  ad_id           BIGINT,
  campaign_id     BIGINT,
  advertiser_id   BIGINT,
  creative_id     BIGINT,
  ad_slot_id      STRING,
  ad_format       STRING,    -- banner/feed/splash
  os              STRING,
  city            STRING,
  province        STRING,
  network         STRING,    -- wifi/4g/5g
  bid_price       DECIMAL(10,4),
  win_price       DECIMAL(10,4),
  ecpm            DECIMAL(10,4),
  pctr            DOUBLE,
  pcvr            DOUBLE,
  is_valid        INT        -- 反作弊标记: 1=有效, 0=无效
)
PARTITIONED BY (dt STRING)
STORED AS ORC;

-- ETL 逻辑:
-- 1. 从 ODS 解析 JSON
-- 2. 关联维度表 (广告信息、广告主信息)
-- 3. 数据清洗 (去重、格式统一)
-- 4. 反作弊标记
```

#### DWS 层 (汇总数据)

```sql
-- DWS: 广告效果日汇总表
CREATE TABLE dws_ad_performance_daily (
  dt              STRING,
  campaign_id     BIGINT,
  advertiser_id   BIGINT,
  ad_slot_id      STRING,
  os              STRING,
  city            STRING,
  -- 指标
  impressions     BIGINT,
  clicks          BIGINT,
  conversions     BIGINT,
  cost            DECIMAL(12,2),
  revenue         DECIMAL(12,2),
  -- 衍生指标
  ctr             DOUBLE,
  cvr             DOUBLE,
  ecpm            DOUBLE,
  cpa             DOUBLE
)
PARTITIONED BY (dt STRING)
STORED AS ORC;

-- SQL:
INSERT OVERWRITE TABLE dws_ad_performance_daily PARTITION (dt = '${date}')
SELECT
  '${date}' as dt,
  i.campaign_id,
  i.advertiser_id,
  i.ad_slot_id,
  i.os,
  i.city,
  COUNT(DISTINCT i.request_id) as impressions,
  COUNT(DISTINCT c.request_id) as clicks,
  COUNT(DISTINCT cv.request_id) as conversions,
  SUM(i.win_price) / 1000 as cost,
  SUM(cv.revenue) as revenue,
  COUNT(DISTINCT c.request_id) / COUNT(DISTINCT i.request_id) as ctr,
  COUNT(DISTINCT cv.request_id) / NULLIF(COUNT(DISTINCT c.request_id), 0) as cvr,
  SUM(i.win_price) / COUNT(DISTINCT i.request_id) * 1000 as ecpm,
  SUM(i.win_price) / NULLIF(COUNT(DISTINCT cv.request_id), 0) / 1000 as cpa
FROM dwd_ad_impression i
LEFT JOIN dwd_ad_click c ON i.request_id = c.request_id AND c.dt = '${date}'
LEFT JOIN dwd_ad_conversion cv ON c.click_id = cv.click_id AND cv.dt = '${date}'
WHERE i.dt = '${date}' AND i.is_valid = 1
GROUP BY i.campaign_id, i.advertiser_id, i.ad_slot_id, i.os, i.city;
```

#### ADS 层 (应用数据)

```sql
-- ADS: 广告主报表 (面向产品/API)
CREATE TABLE ads_advertiser_report (
  dt              STRING,
  advertiser_id   BIGINT,
  advertiser_name STRING,
  total_cost      DECIMAL(12,2),
  total_impressions BIGINT,
  total_clicks    BIGINT,
  total_conversions BIGINT,
  avg_ctr         DOUBLE,
  avg_cvr         DOUBLE,
  avg_cpa         DOUBLE,
  roas            DOUBLE
)
PARTITIONED BY (dt STRING);
```

### DIM 维度表

```sql
-- 广告计划维度表
CREATE TABLE dim_campaign (
  campaign_id     BIGINT,
  campaign_name   STRING,
  advertiser_id   BIGINT,
  advertiser_name STRING,
  industry        STRING,    -- 行业
  budget_daily    DECIMAL(12,2),
  bid_type        STRING,    -- cpc/cpm/ocpm
  target_cpa      DECIMAL(10,2),
  status          STRING,    -- active/paused/deleted
  create_time     TIMESTAMP,
  update_time     TIMESTAMP
);

-- 广告位维度表
CREATE TABLE dim_ad_slot (
  ad_slot_id      STRING,
  slot_name       STRING,
  app_id          STRING,
  app_name        STRING,
  slot_type       STRING,    -- banner/feed/splash/interstitial
  slot_size       STRING,    -- 640x100
  floor_price     DECIMAL(10,4)
);
```

---

## 计算引擎

### Hive

```
适用场景:
  - 大规模 ETL (TB 级)
  - 复杂 SQL 查询
  - 数仓建设

优点: SQL 友好、生态成熟、稳定
缺点: 延迟高 (分钟级)、不适合迭代计算

执行引擎:
  - MapReduce (已过时)
  - Tez (中等性能)
  - Spark (推荐，性能最好)

SET hive.execution.engine=spark;
```

### Spark

```
适用场景:
  - 复杂 ETL
  - 特征计算
  - 模型训练样本生成
  - 迭代计算 (ML)

优点: 内存计算快、API 丰富、支持 ML
缺点: 内存消耗大、调优复杂

Spark SQL 示例:
val impressions = spark.read.table("dwd_ad_impression").filter($"dt" === "2024-01-15")
val clicks = spark.read.table("dwd_ad_click").filter($"dt" === "2024-01-15")

val result = impressions
  .join(clicks, Seq("request_id"), "left")
  .groupBy("campaign_id", "os")
  .agg(
    count("*").as("impressions"),
    count(clicks("request_id")).as("clicks"),
    sum("win_price").as("cost")
  )
```

### Presto / Trino

```
适用场景:
  - 交互式查询 (秒级响应)
  - Ad-hoc 分析
  - 跨数据源查询

优点: 查询速度快、支持多数据源
缺点: 不适合大规模 ETL、内存限制

典型用法:
  数据分析师通过 Presto 查询 Hive 表
  运营人员通过 BI 工具 (Superset/Metabase) 连接 Presto
```

---

## 调度系统

### Airflow

```python
# 广告数仓 DAG 示例
from airflow import DAG
from airflow.providers.apache.hive.operators.hive import HiveOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': True,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'ad_data_warehouse',
    default_args=default_args,
    schedule_interval='0 3 * * *',  # 每天凌晨3点
    start_date=datetime(2024, 1, 1),
)

# Task 1: ODS → DWD
ods_to_dwd_impression = HiveOperator(
    task_id='ods_to_dwd_impression',
    hql='sql/dwd_ad_impression.sql',
    hive_cli_conn_id='hive_default',
    dag=dag,
)

# Task 2: ODS → DWD (点击)
ods_to_dwd_click = HiveOperator(
    task_id='ods_to_dwd_click',
    hql='sql/dwd_ad_click.sql',
    dag=dag,
)

# Task 3: DWD → DWS
dwd_to_dws = HiveOperator(
    task_id='dwd_to_dws_performance',
    hql='sql/dws_ad_performance_daily.sql',
    dag=dag,
)

# Task 4: DWS → ADS
dws_to_ads = HiveOperator(
    task_id='dws_to_ads_report',
    hql='sql/ads_advertiser_report.sql',
    dag=dag,
)

# 依赖关系
[ods_to_dwd_impression, ods_to_dwd_click] >> dwd_to_dws >> dws_to_ads
```

### 调度系统对比

| 系统 | 特点 | 适用场景 |
|------|------|---------|
| **Airflow** | Python DAG、社区活跃 | 通用调度 |
| **DolphinScheduler** | 国产、可视化强 | 国内企业 |
| **Azkaban** | LinkedIn 开源、简单 | 中小规模 |
| **Oozie** | Hadoop 原生 | Hadoop 生态 |

---

## 数据质量

### 数据校验规则

```sql
-- 数据量校验: 与前一天对比
SELECT
  '${date}' as dt,
  COUNT(*) as today_cnt,
  LAG(COUNT(*)) OVER (ORDER BY dt) as yesterday_cnt,
  (COUNT(*) - LAG(COUNT(*)) OVER (ORDER BY dt))
    / LAG(COUNT(*)) OVER (ORDER BY dt) as change_rate
FROM dwd_ad_impression
WHERE dt IN ('${date}', '${yesterday}')
GROUP BY dt
HAVING ABS(change_rate) > 0.3;  -- 变化超过30%告警

-- 空值校验
SELECT
  COUNT(*) as total,
  SUM(CASE WHEN user_id IS NULL THEN 1 ELSE 0 END) as null_user_cnt,
  SUM(CASE WHEN ad_id IS NULL THEN 1 ELSE 0 END) as null_ad_cnt
FROM dwd_ad_impression
WHERE dt = '${date}';

-- 数据对账: 实时 vs 离线
SELECT
  a.dt,
  a.impressions as realtime_imp,
  b.impressions as offline_imp,
  ABS(a.impressions - b.impressions) / b.impressions as diff_rate
FROM realtime_report a
JOIN dws_ad_performance_daily b ON a.dt = b.dt
WHERE ABS(diff_rate) > 0.01;  -- 差异超过1%告警
```

### 数据血缘 (Data Lineage)

```
追踪数据从源到目标的流转路径:

ads_advertiser_report
  ← dws_ad_performance_daily
    ← dwd_ad_impression + dwd_ad_click + dwd_ad_conversion
      ← ods_ad_request_log + ods_ad_click_log + ods_ad_conversion_log
        ← Kafka (ads.impression.server + ads.click.server + ads.conversion.s2s)

用途:
  - 问题排查: 报表数据异常时快速定位源头
  - 影响分析: 修改上游表时评估下游影响
  - 合规审计: 追踪数据的使用和流转
```

---

## 广告离线计算典型任务

| 任务 | 频率 | 引擎 | 说明 |
|------|------|------|------|
| **日志入仓** | 每小时 | Spark | Kafka → ODS |
| **DWD ETL** | T+1 | Hive/Spark | ODS → DWD |
| **DWS 聚合** | T+1 | Hive/Spark | DWD → DWS |
| **报表生成** | T+1 | Hive/Spark | DWS → ADS |
| **用户画像** | T+1 | Spark | 标签计算 |
| **样本生成** | T+1 | Spark | 训练样本 |
| **特征回填** | T+1 | Spark | 离线特征 |
| **数据导出** | T+1 | Sqoop/DataX | Hive → MySQL/ClickHouse |
| **数据对账** | T+1 | Hive | 多方数据对比 |

---

## 与大数据开发的日常工作

- **数仓建模**: 设计和维护广告数仓的分层架构
- **ETL 开发**: 编写 Hive SQL / Spark 任务
- **调度管理**: 配置和维护调度 DAG
- **数据质量**: 数据校验规则的开发和维护
- **性能优化**: SQL 调优、数据倾斜处理、小文件合并
- **需求响应**: 新报表、新指标的开发

---

## 面试高频问题

1. 数据仓库分层架构 (ODS/DWD/DWS/ADS) 各层的职责？
2. 广告数仓的核心表有哪些？如何设计？
3. Hive 和 Spark 的区别？各自适用什么场景？
4. 如何保证离线数据质量？
5. 数据倾斜如何处理？
6. 调度系统如何保证任务的依赖和重试？

---

## 推荐阅读

- 《数据仓库工具箱》— Kimball
- 《大数据之路：阿里巴巴大数据实践》
- [Apache Spark 官方文档](https://spark.apache.org/)
- [Apache Airflow 官方文档](https://airflow.apache.org/)
