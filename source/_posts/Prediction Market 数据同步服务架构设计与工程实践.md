---
title: Prediction Market 数据同步服务架构设计与工程实践
date: 2026-06-07 20:10:00
tags: [web3, polymarket, prediction-market, go, architecture]
---

## 前言

Prediction Market（预测市场）是 Web3 领域重要的应用场景之一。Polymarket 作为最大的去中心化预测市场平台，提供 Gamma、Data、CLOB 三套 API 接口供开发者消费数据。

但要基于 Polymarket 数据构建一个面向用户的前端应用，需要解决几个实际问题：

- **数据获取**：API 有频率限制，单次查询量有限，需要周期性同步
- **数据一致性**：多 API 之间的数据需要关联和校验，避免状态不一致
- **实时性**：用户需要看到市场的最新成交和结果，依赖 Kafka 事件流
- **搜索能力**：PostgreSQL 不适合全文搜索，需要用 Elasticsearch 做聚合查询
- **性能**：API 请求和数据库查询需要缓存层支撑

本文介绍的这套系统正是为解决这些问题而生的——一个纯后端 Worker 服务，负责从 Polymarket 拉取数据、持久化存储、实时处理事件，最终为前端提供一致的、高性能的数据底座。

## 一、整体架构

系统是一个**纯 Worker 架构**——没有 HTTP API，没有前端，只有一个后台进程持续运行。

```mermaid
flowchart LR
    subgraph Input [数据源]
        G[Polymarket Gamma API<br/>事件/市场/标签/翻译]
        D[Polymarket Data API<br/>排行榜/持仓/交易]
        C[Polymarket CLOB API<br/>订单簿/价格]
        K[Kafka<br/>链上事件流]
    end

    subgraph Worker [Worker 服务]
        J[定时调度器<br/>22 个同步Job]
        CON[Kafka Consumer<br/>实时事件消费]
    end

    subgraph Storage [存储层]
        PG[(PostgreSQL<br/>prediction schema)]
        ES[(Elasticsearch<br/>web3_prediction 索引)]
        RD[(Redis<br/>缓存/队列)]
    end

    Input --> Worker
    Worker --> Storage
    Storage --> FE
```

### 核心启动流程

Worker 的启动入口在 `cmd/worker/main.go`，启动顺序如下：

```mermaid
flowchart TD
    Start[启动 Worker]
    Config[加载配置<br/>Viper + ENV Override]
    OTel[初始化 OpenTelemetry Trace]
    Logger[初始化 Zap 日志 + Lumberjack 轮转]
    Repo[初始化仓储层<br/>PG / ES / Redis / Kafka]
    Scheduler[启动定时调度器<br/>22 个 Job 注册]
    Consumers[启动 Kafka Consumer<br/>OrderConsumer]
    Metrics[启动 Prometheus Metrics<br/>:8091/metrics]
    Block[Block 等待 SIGINT/SIGTERM]
    Shutdown[优雅关闭]

    Start --> Config --> OTel --> Logger --> Repo
    Repo --> Scheduler
    Repo --> Consumers
    Scheduler --> Metrics
    Consumers --> Metrics
    Metrics --> Block --> Shutdown
```

### 配置管理

系统使用 Viper 管理配置，支持多环境隔离：

```yaml
# config/config.worker.yaml 核心配置
kafka:
  brokers:
    - "alibaba-kafka-broker:9092"
  topic:
    prediction_events: "web3_prediction_market_events"
    trade_events: "web3_prediction_trade_events"
  consumer_group: "web3_prediction_data"

redis:
  address: "localhost:6379"
  db: 3                # 主缓存 DB
  metrics_db: 4        # 监控指标 DB

postgres:
  dsn: "postgres://user:pass@localhost:5432/prediction?sslmode=disable"

elasticsearch:
  addresses:
    - "http://alibaba-es:9200"
  index: "web3_prediction"
  bulk_worker_num: 16
  bulk_flush_bytes: 5242880  # 5MB
  bulk_flush_interval: 3s

worker:
  trade_consumer_count: 64
  prediction_consumer_count: 32

limiter:
  trade_per_second: 3000
  prediction_per_second: 3000
```

敏感字段通过 `MOONX_*` 环境变量覆盖，不硬编码在配置文件中。

## 二、数据同步体系

系统运行 22 个同步 Job，覆盖不同类型的数据同步需求。

### 2.1 Job 分类

系统共运行 22 个同步 Job，按类别划分：

| 类别 | Job 名称 | 调度方式 | 职责 |
|------|----------|----------|------|
| **事件同步** | `new_event` | 5min 定时 | 扫描新事件 |
| | `active_event_sync` | 10min 定时 | 活跃事件增量更新 |
| | `closed_event` | 6h 定时 | 已关闭事件刷新 |
| | `date_sync` | 12h 定时 | 按日期范围回扫 |
| **行情同步** | `price_job` | 3min 定时 | 市场价格同步 |
| | `crypto_sync` | 5min 定时 | 加密货币上下对 |
| **质量修复** | `outcome_fix` | 30min 定时 | 修复畸形 outcome |
| | `stale_status_sync` | 4h 定时 | 修复状态不一致 |
| | `slug_check_sync` | 4h 定时 | 检测失效 slug |
| | `logo_audit` | 6h 定时 | Logo 图片审核 |
| **排行榜** | `top_list_sync` | 3h 定时 | Top 榜单修复 |
| | `trend_list_sync` | 1h 定时 | 趋势/最新/热门榜单 |
| **ES 同步** | `es_sync` | 5s 定时 | PG→ES 增量游标同步 |
| | `es_empty_markets_clean` | 10min 定时 | 清理空市场的文档 |
| **翻译** | `translation_sync` | 2h 定时 | 全量翻译同步 |
| | `recent_translation_sync` | 20min 定时 | 近期事件翻译 |
| | `new_event_translation_sync` | 10min 定时 | 新事件翻译 |
| **聪明钱** | `pm_smart_money` | 启动时一次 | 聪明钱榜单/持仓/交易 |
| | `pm_sm_bgwin` | 启动时一次 | 最大赢家 |
| | `pm_sm_logo` | 启动时一次 | 聪明钱图片审核 |
| **常驻修缮** | `missing_slug_backfill` | 内部轮询 5s | 事件 slug 回填 |
| | `pending_updown_price` | 内部轮询 5s | 上下对价格回填 |

> 注：`pm_smart_money` 等注册为 "一次性" 的 Job 仅在启动时执行一次，不周期性重复运行。`missing_slug_backfill` 虽注册为一次性，但其 Run 方法内部包含 `for { runOnce(); time.Sleep(5s) }` 循环，实际常驻运行。

### 2.2 自定义调度器

系统没有使用 Cron，而是实现了自己的 `Scheduler`：

```go
// scheduler 核心：全局并发控制，32 goroutine worker pool
type Scheduler struct {
    globalSemaphore *GlobalJobSemaphore  // 最大 10 个并发 job
    workerPool      *pool.Pool           // 32 goroutines
}

// 注册周期性 Job
scheduler.RegisterJob("active_event_sync", 10*time.Minute, j.ActiveEventSync)

// 注册一次性 Job（启动时执行一次，但任务内部可能自带循环）
scheduler.RegisterOnceJob("missing_slug_backfill", 30*time.Second, j.MissingSlugBackfill, 0)
```

关键设计点：
- **Semaphore**：全局限制最大并发数，防止突发 Job 打满资源
- **Idempotency**：同一 Job 正在执行时，下次触发自动跳过，不堆积
- **Timeout**：Job 超时为 interval 本身，截断在 [10s, 30min] 范围内，防止死等
- **Worker Pool**：ticker 触发后在 32 goroutine 池中执行，避免阻塞调度循环
- **Prometheus Metrics**：所有 Job 的执行耗时、失败次数、超时次数都有监控

### 2.3 事件同步主流程

以 `active_event_sync` 为例，展示一个典型 Job 的数据流动：

```mermaid
flowchart TD
    Trigger[定时触发 10min]
    Fetch[请求 Gamma API<br/>GET /events?closed=false&limit=100]
    Convert[Gamma->Model 转换<br/>converter.go]
    Upsert[GORM Upsert<br/>prediction.pm_events]
    MarketSync[同步关联市场<br/>prediction.pm_markets]
    Cache[更新 Redis 缓存<br/>event:slug -> market bundle]
    Slug[提取 slug + locales<br/>写入 Redis sets]

    Trigger --> Fetch
    Fetch --> Convert
    Convert --> Upsert
    Upsert --> MarketSync
    MarketSync --> Cache
    Cache --> Slug
```

### 2.4 Gamma 客户端的增强设计

Gamma API 客户端不止是一个 HTTP 封装。数据返回后会经过多层增强管道：

**转换流水线 `convertGammaEvents`**

```go
func (c *GammaAPIClient) convertGammaEvents(ctx, gammaEvents) (events, markets) {
    for each event in gammaEvents:
        markets = append(ConvertToMarket(market))
        events = append(ConvertToEvent(event))

    c.EventExtraConditionIDs(events, markets)   // 建立 event → condition_ids 关联
    c.applyRedisOutcomes(ctx, markets)           // 从 Redis 回填 outcome
    c.applyRedisQuestionID(ctx, markets)         // 从 Redis 回填 tx_hash、log_index
    reconcileEventMarketStatus(events, markets)  // 保证 event / market 状态一致
    return events, markets
}
```

**1. 从 SSG 页面提取链上数据**

在 `FetchEventsBySlug` / `FetchEventsBySlugs` 路径中，客户端额外调用 `EventExtraChainInfo`，通过请求 Polymarket 的 Next.js SSG 构建产物 `/_next/data/build-{id}/en/event/{slug}.json`，从脱水状态中反序列化出 `TransactionHash` 和 `LogIndex`，补充到 market 的 `ChainInfo`：

```go
func (c *GammaAPIClient) EventExtraChainInfo(ctx, slug, events, markets) {
    pageProps, _ := c.FetchPagePropsBySlug(ctx, slug)        // GET /_next/data/...
    ssgMarkets, _ := c.MarketsFromSSGPayload(pageProps)      // 从 dehydratedState 中提取
    for _, market := range markets {
        if match := findMatch(market.ConditionID, ssgMarkets); match != nil {
            market.ChainInfo.TxHash = match.TransactionHash
            market.ChainInfo.EventIndex = match.LogIndex
        }
    }
}
```

这是在不依赖区块链 RPC 节点的情况下获取链上决议数据的巧妙方法。

**2. 从 Redis Pipeline 回填缺失数据**

使用 Redis Pipeline 批量查询 `outcome` 和 `question_id` / `tx_hash`，避免 Gamma API 返回数据中这些字段为空：

```go
func (c *GammaAPIClient) applyRedisOutcomes(ctx, markets) {
    pipe := c.repo.GetRedis().Pipeline()
    for _, m := range markets {
        cmds[i] = pipe.HGet(ctx, key, "market_outcome")
    }
    pipe.Exec(ctx)
    // 非空 outcome 回填并标记 market 为 closed
}
```

**3. 状态一致性修正**

`reconcileEventMarketStatus` 保证 event 与 market 状态不矛盾：

- event 已 closed → 其下所有未关闭的 market 强制 closed
- event 仍 open 但所有 market 已 closed（或 inactive）→ event 提升为 closed

## 三、实时事件处理

### 3.1 Kafka 消费者架构

系统使用通用的泛型消费者框架 `Consumer[T]`：

```go
type Consumer[T any] struct {
    kafkaReader  *kafka.Reader     // Kafka 消费者
    buffers      []chan T          // 每个 worker 独立缓冲 channel
    limiter      *rate.Limiter     // 自适应速率限制
    workerSize   int               // 工作协程数
    workerLoadMap []int64          // 各 worker 负载状态
}
```

支持的特性：
- **Worker Pool**：多协程并发消费，通过 channel 分发
- **Hash Routing**：相同 key 路由到同一协程，保证有序处理
- **Adaptive Rate Limiting**：根据 buffer 堆积程度动态调节消费速率
- **Graceful Shutdown**：等待所有 in-flight 消息处理完成

### 3.2 事件处理链路

```mermaid
flowchart TD
    subgraph Kafka [Kafka Topic]
        OF[ORDER_FILLED]
        OM[ORDERS_MATCHED]
        CR[CONDITION_RESOLUTION]
        QI[QUESTION_INITIALIZED]
    end

    subgraph Handler [OrderHandler]
        Resolve[从 Redis 解析 event_slug<br/>通过 tokenAssetId 映射]
        Transform[数据转换<br/>填充 trade_time / side / size]
        Forward[转发到 trade_events topic]
    end

    subgraph Resolution [决议处理]
        Close[关闭市场/事件]
        Outcome[确定 outcome]
        UpdatePG[更新 PostgreSQL<br/>event + market 状态]
        UpdateES[更新 Elasticsearch<br/>同步状态变更]
        UpdateCache[更新 Redis 缓存<br/>清除旧热点数据]
    end

    OF --> Resolve
    OM --> Resolve
    CR --> Close
    QI --> Store[写入 Redis<br/>question_id -> slug 映射]

    Resolve --> Transform
    Transform --> Forward
    Close --> Outcome
    Outcome --> UpdatePG
    UpdatePG --> UpdateES
    UpdateES --> UpdateCache
```

### 3.3 CONDITION_RESOLUTION 深度处理

当市场条件被决议时，`handleUma` 的实际流程：

```
1. 从 Kafka 消息中解析 question_id
2. 根据 question_id JOIN pm_markets 找到关联的 event
3. 遍历 event.Markets，匹配 condition_id
4. 通过 PayoutNumerators 确定 outcome：
   - payoutNumerator == 1.0 → 索引位对应的 outcome 胜出
   - 更新 market.Outcome = YesLabel / NoLabel
5. 查询 Gamma API 确认事件最新状态
6. BatchUpsert markets → PostgreSQL（优先写入 PG）
7. 所有 market 已 closed → 更新 event.Status = closed
   - 同时触发 updown 价格回填
8. SaveToEs → Elasticsearch（BulkIndexer 异步）
9. BatchHSet → Redis（更新 market 缓存）
```

注意写入顺序：**先写 PG，再写 ES**。ES 写入通过 BulkIndexer 异步执行，失败时 `es_sync` 会基于 updated_at 游标兜底重试。

## 四、存储层设计

### 4.1 PostgreSQL 数据模型

所有表位于 `prediction` schema 下，使用 `pm_` 前缀：

```mermaid
erDiagram
    pm_events ||--o{ pm_markets : contains
    pm_events {
        string slug PK
        string platform PK
        string title
        string description
        jsonb tags
        datetime start_date
        datetime end_date
        numeric volume
        numeric liquidity
        string status
    }
    pm_markets {
        string condition_id PK
        string yes_token
        string no_token
        string question
        string outcome
        jsonb prices
        jsonb chain_info
        jsonb fee_schedule
        jsonb title_locale
        jsonb question_locale
        string status
    }
    pm_trades {
        bigint id PK
        string tx_hash
        string proxy_wallet
        string side
        numeric size
        numeric price
        datetime block_timestamp
    }
    pm_tasks {
        string name PK
        string state
        jsonb data
    }
```

事件和市场是一对多的关系。一个事件（如"2026 年美国总统大选"）可以包含多个市场（如"谁将赢得民主党提名"、"谁将赢得共和党提名"）。

### 4.2 Elasticsearch 搜索索引

ES 索引 `web3_prediction` 采用**反范式化**文档模型——每个事件文档内嵌其所有市场：

```json
{
  "id": "event-slug-123",
  "slug": "will-bitcoin-hit-100k",
  "title": "Will Bitcoin hit $100,000 in 2026?",
  "description": "Prediction market for Bitcoin price...",
  "logo": "https://...",
  "tags": ["crypto", "bitcoin"],
  "startDate": "2026-01-01T00:00:00Z",
  "endDate": "2026-12-31T00:00:00Z",
  "status": "open",
  "volume": 1500000.00,
  "liquidity": 500000.00,
  "markets": [
    {
      "condition_id": "0xabc...",
      "question": "Will Bitcoin hit $100K?",
      "outcome": null,
      "prices": {"yes": 0.45, "no": 0.55},
      "yes_token": "0x123...",
      "no_token": "0x456..."
    }
  ]
}
```

索引配置：3 shard、1 replica，以支持高效的全文搜索和聚合查询。

### 4.3 Redis 缓存体系

```
DB 0: top_list（榜单缓存）
DB 1: native_price（链上原生价格）
DB 3: main（事件/市场缓存）
DB 4: metrics（监控指标）

Key 模式：
  event:slug:{slug}           → EventCache（事件+市场 Bundle）
  event:slug:set              → 所有已知 slug 的集合
  event:locale:slug:set       → 需要翻译的 slug 集合
  outcome:{condition_id}      → 市场 outcome 缓存
  question_id:{question_id}   → question_id → slug 映射
```

### 4.4 ES 增量同步核心机制

`es_sync` Job 每 5 秒运行一次，使用游标分页实现增量同步：

```mermaid
flowchart TD
    Start["es_sync 触发"]
    GetCursor["读取游标<br/>LatestUpdatedAt + LatestID"]
    QueryPG["查询 PG<br/>updated_at&gt;cursor OR<br/>(updated_at=cursor AND id&gt;cursor)<br/>limit 1000"]
    Check{有数据?}
    Cache["EventCache 命中?"]
    PGFallback["从 PG 按 slug 回退<br/>每批 100 条"]
    Convert["转换 Event→ES Document<br/>含嵌套 markets"]
    Bulk["构建 Bulk Index 请求"]
    WriteES["Bulk 写入<br/>Elasticsearch"]
    UpdateCursor["更新游标<br/>取最后一条记录"]
    End["等待 5s 下次触发"]

    Start --> GetCursor --> QueryPG --> Check
    Check -- 是 --> Cache
    Cache -- 命中 --> Convert
    Cache -- 未命中 --> PGFallback --> Convert
    Convert --> Bulk --> WriteES --> UpdateCursor
    UpdateCursor --> GetCursor
    Check -- 否 --> End
```

这种方式避免了全量扫描，每次只同步新增或修改的事件，效率高且可追踪。

## 五、聪明钱追踪系统

Smart Money（聪明钱）是 Polymarket 上的高收益交易者。系统通过 Data API 追踪他们的行为：

```mermaid
flowchart TD
    subgraph API [Polymarket Data API]
        LB[Leaderboard<br/>按类别/周期/排序]
        Profile[Wallet Profile<br/>PNV/Volume/Wins]
        Positions[Positions<br/>持仓明细]
        Trades[Trades<br/>交易记录]
    end

    subgraph PG [PostgreSQL]
        R[pm_sm_ranking<br/>排行榜]
        A[pm_sm_address<br/>钱包画像]
        P[pm_sm_position<br/>持仓快照]
        ACT[pm_sm_activity<br/>活动流水]
    end

    subgraph ES [Elasticsearch]
        INDEX[smart_money 索引]
    end

    LB --> R
    Profile --> A
    Positions --> P
    Trades --> ACT

    A --> INDEX
    P --> INDEX
```

### 排行榜同步

Data API 的排行榜支持多维度组合：

- **Categories**：`all`、`prediction`、`restricted`、`sports`
- **TimePeriods**：`all_time`、`7d`、`30d`、`365d`
- **OrderBy**：`pnl`、`volume`、`wins`、`accuracy`
- **RankingTypes**：`absolute`、`relative`、`roi`

系统会遍历所有组合，每批拉取 5 条，最多同步到 Top 200 的钱包地址。

### 持仓快照

每个聪明钱钱包的持仓包括：
- 市场标题、outcome 方向
- 持仓数量、均价、当前盈亏
- 关联的交易明细

数据通过 Data API 的 `getLeaderboardPosition` 接口获取，按 offset 分页拉取。

## 六、图片审核流水线

由于平台展示的需要，事件和市场的 Logo 图片需要进行审核。系统通过 MoonX 内部服务完成：

```mermaid
flowchart LR
    New[新事件发现]
    Queue[写入 Redis<br/>logo_audit:queue]
    Audit[MoonX API<br/>POST /audit-image]
    Check{审核通过?}
    Tag[标记 logo_audited=true]
    Manual[人工审核队列]

    New --> Queue
    Queue --> Audit
    Audit --> Check
    Check -- 是 --> Tag
    Check -- 否 --> Manual
```

MoonX 的域名 `uploads.bydfi.in` 的图片视为已审核，直接跳过。

## 七、异常检测与数据修复

### 7.1 状态不一致检测

系统通过 `fix-status` CLI 工具提供手动修复能力：

```
# 检查指定事件的状态一致性
fix-status check --slugs "will-btc-hit-100k,will-eth-merge"

# 自动修复
fix-status check --slugs "will-btc-hit-100k" --fix

# 批量关闭过期事件
fix-status closed --days 7 --dry-run
fix-status closed --days 7  # 实际执行
```

### 7.2 自动修复机制

几个 Job 专职处理数据质量：

| Job | 检测内容 | 修复方式 |
|-----|---------|---------|
| `outcome_fix` | Market outcome 格式异常 | 根据 prices 推断并修正 |
| `stale_status_sync` | 事件已在链上结束但状态仍为 open | 从 Gamma 获取最新状态并更新 |
| `slug_check_sync` | Gamma 返回 404 的 slug | 从订单数据中重新发现正确 slug |
| `es_empty_markets_clean` | ES 文档中 markets 数组为空 | 从 PG 回填或删除 |

### 7.3 数据不一致的根源

数据不一致主要来自三个方面：

1. **API 延迟**：Gamma API 的事件状态更新晚于链上决议
2. **并发写入**：多个 Job 可能同时修改同一条记录
3. **外部变更**：事件被管理员手动修改，Job 需要重新同步

应对策略是用**定期修复 Job** 兜底，而非在写入链路加复杂约束。

## 八、工程取舍与设计权衡

| 取舍点 | 选择 | 理由 |
|--------|------|------|
| **架构模式** | 纯 Worker | 无 HTTP 端口暴露，降低攻击面；所有任务可调度和监控 |
| **调度器** | 自研而非 Cron | 支持 idempotency、semaphore、timeout、metrics 一整套 |
| **消费者** | 泛型 Consumer[T] | 一套框架处理多种 Kafka 消息类型，避免重复代码 |
| **ES 同步** | 游标增量同步 | 避 full reindex，5s 延迟可接受 |
| **状态一致性** | 定期修复兜底 | 不追求强一致，最终一致性 + 自动修复 |
| **链上数据获取** | SSG 页面解析而非 RPC | 无需区块链节点，降低基础设施成本 |
| **翻译同步** | 三级频率（2h/20min/10min） | 全量保底 + 增量及时 + 新事件优先 |
| **图片审核** | 外部服务 + 白名单 | 预审核域名跳过，减少 API 调用 |
| **配置管理** | Viper + ENV override | 支持多环境无侵入覆盖，敏感信息不落文件 |
| **监控** | Prometheus | 每个 Job 和 Consumer 都有完整 metric，可对接 Grafana |

## 九、运维与监控

### 启动方式

```bash
# 启动 Worker
make worker

# 启动修复工具
go run cmd/fix-status/main.go check --slugs "event-slug"

# 清理构建
make clean
```

### 可观测性

- **Prometheus Metrics**（`:8091/metrics`）
  - `job_duration_seconds`：Job 执行耗时分布
  - `job_failures_total`：Job 失败计数
  - `kafka_messages_processed_total`：Kafka 处理总量
  - `es_bulk_bytes_total`：ES 写入流量
- **Health Check**（`:8091/healthz`）
- **OpenTelemetry Tracing**：链路追踪，定位瓶颈
- **Zap 日志** + **Lumberjack 轮转**：日志文件大小管理

### 优雅关闭

Worker 在收到 SIGINT/SIGTERM 时：
1. 停止接受新的 Kafka 消息
2. 等待所有 in-flight job 完成
3. 关闭数据库连接池
4. 关闭 Kafka reader/writer
5. 退出进程

---

以上是 Prediction Market 数据同步服务的完整架构设计。系统通过 22 个同步 Job 配合 Kafka 实时消费，实现了从 Polymarket 多 API 到 PostgreSQL/Elasticsearch/Redis 的全量数据同步链，为前端预测市场应用提供了可靠的数据基础设施。
