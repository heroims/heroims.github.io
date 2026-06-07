---
title: Web3 代币数据服务 — 架构设计与工程实践
date: 2026-06-06 20:10:00
tags: [web3, go, architecture, blockchain, solana, evm]
---

## 前言

在 Web3 领域，代币数据是上层应用的基础。无论是行情展示、交易决策、安全风控，还是智能投研，都离不开准确、实时的代币元数据、价格、持有者分布和安全信息。但搭建一个覆盖多链、实时更新的代币数据服务面临诸多挑战：

**多链异构**：Solana 和 EVM 链（BSC、Base）的链上数据结构、RPC 接口、代币标准完全不同。Solana 使用 SPL Token 标准，通过 PDA 派生账户地址；EVM 链使用 ERC-20 标准，合约状态存储在全局 state trie 中。两者在元数据读取、持有者查询、交易解析等各个层面都有本质差异。

**数据来源多样**：链上 RPC 提供原始数据，第三方 API（CoinGecko、CoinMarketCap、GoPlus、DexScreener）提供增强信息，链下数据源（Twitter、Discord、Telegram 社交链接）补充元数据。多源汇聚带来了数据冲突、时效不一致、部分缺失等问题。

**实时性要求高**：从代币创建 → 首笔交易 → 市场数据更新，需要在秒级完成。尤其是 Meme 代币的"内盘"阶段，价格波动极快，数据延迟直接导致用户错过交易窗口。

**外部依赖脆弱**：数十个外部 API 各有不同的速率限制（QPS）、认证方式、响应格式。部分 API 还有反爬机制和 IP 白名单要求，容错降级是刚需。

本文介绍一个用 Golang 构建的生产级代币数据服务，已覆盖 Solana、BSC、Base 三条链，对接数十个外部数据源。下面从架构设计、核心技术、数据流和工程实践四个维度展开。

## 一、功能概览

该服务的定位是**代币数据中台**，核心职责包括：

| 模块 | 功能 |
|------|------|
| 代币收录 | 监听链上代币创建事件（Mint），自动入库 |
| 元数据补全 | 名称、符号、图标、描述等信息补充 |
| 安全检查 | 貔貅检测、可铸币/可冻结属性、合约安全打分 |
| 持有者分析 | 持有者分布、持仓变化、聪明钱/开发者标记 |
| 市场数据 | 价格、交易量、流动性、FDV、市值、热度评分 |
| 榜单排名 | 热门榜、新币榜、即将完成榜、已完成榜 |
| 池子管理 | DEX 流动性池地址解析、池费查询与更新 |

## 二、系统架构

### 2.1 整体架构

```
                    ┌─ token_events ──────────┐
                    │  trade_events             │
                    │  data_events              │
                    └────────┬──────────────────┘
                             │ Kafka
                    ┌────────▼────────────┐
                    │     Core 编排器      │
                    │  ├─ 3 个 Consumer   │
                    │  ├─ Scheduler       │
                    │  └─ Monitor         │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        TokenUpdater   Scheduler任务    PoolFeeService
              │              │              │
              ▼              ▼              ▼
         Elasticsearch  外部API补全    Redis Cache/Queue
              │
              ▼
         Kafka (下游内盘更新分发)
```

**Core 编排器**在启动阶段依次初始化：

1. **Repository** - 初始化 ES、PostgreSQL、Redis 连接池
2. **ChainManager** - 初始化各链的 RPC 客户端（Solana JSON-RPC、EVM ethclient）
3. **TokenUpdater** - 启动统一缓冲写入器
4. **3 个 Kafka Consumer** - 订阅不同事件主题
5. **Scheduler** - 注册 29+ 个定时任务
6. **Monitor** - 暴露 Prometheus 指标（`:8091/metrics`）

### 2.2 存储选型

| 存储 | 用途 | 选型理由 |
|------|------|----------|
| Elasticsearch | 代币主存储、全文检索 | Schema-less，嵌套对象支持好，TB 级数据聚合查询性能可接受 |
| PostgreSQL | Pairs/Pools 关系型数据 | 强一致、事务支持、JOIN 查询 |
| Redis | 缓存 + 补偿队列 + 分布式锁 | 高性能，List/Set/SortedSet 数据结构适合队列场景 |
| SelectDB(MySQL) | 余额流水、聪明钱分析数据 | 列存分析型引擎，适合时序大表聚合 |

ES 是代币数据的事实来源，所有代币更新最终写入 ES。PG 主要负责关系型数据的结构化查询——例如"查找某个 DEX 的所有活跃 Pool"。Redis 除了缓存还承担两个补偿队列的存储角色。

### 2.3 配置管理

使用 Viper + fsnotify 实现配置热重载：

```go
viper.SetConfigName("config.worker")
viper.SetConfigType("yaml")
viper.AddConfigPath("./config/")
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
    log.Info("Configuration changed, reloading")
    cfg.Reload()
})
```

这种设计使得修改数据源地址、调整速率限制、开关特定任务等操作无需重启进程，避免连接中断。

### 2.4 Elasticsearch 存储模型

ES 是代币数据的**唯一持久存储**（PG 的 token 写入已禁用），所有代币的增删改查都围绕 ES 进行。

#### 文档结构

索引名 `web3_tokens`，3 分片 1 副本。每条文档对应一个代币，文档 ID 为 `{network}_{address}`（如 `SOLANA_Abc123...`、`BASE_0xabcd...`）。

核心字段结构（Token 模型的 JSON 序列化）：

```go
type Token struct {
    Network           string        `json:"network"`            // 链名称
    ChainID           int           `json:"chain_id"`           // 链 ID
    Address           string        `json:"address"`            // 代币地址
    AddressNormalized string        `json:"address_normalized"` // 归一化地址
    Symbol            string        `json:"symbol"`             // 交易对符号
    Name              string        `json:"name"`               // 代币名称（text 类型）
    Decimals          int           `json:"decimals"`
    PriceUSD          float64       `json:"price_usd"`
    TotalSupply       float64       `json:"total_supply"`
    HeatScore         HeatScore     `json:"heat_score"`         // 热度分 {m5,h1,h6,h24}
    IsInner           bool          `json:"is_inner"`           // 是否内盘阶段
    Tags              []string      `json:"tags"`               // 标签列表
    // 嵌套对象
    MarketInfo        *MarketInfo   `json:"market_info"`        // 行情（各窗口量价）
    PoolInfo          PoolInfo      `json:"pool_info"`          // 池子（含 nested pairs）
    SecurityInfo      *SecurityInfo `json:"security_info"`      // 安全审计
    HolderInfo        *HolderInfo   `json:"holder_info"`        // 持有者分布
    InnerPool         InnerPool     `json:"inner_pool"`         // 内盘信息
    SocialInfo        SocialInfo    `json:"social_info"`        // 社交链接
    ContractInfo      ContractInfo  `json:"contract_info"`      // 合约信息
    // ... 更多
}
```

Mapping 要点：

| 字段 | ES 类型 | 说明 |
|------|---------|------|
| `name` | `text` + `.keyword` | 支持全文搜索和精确匹配 |
| `tags` | `text` + `.keyword` | 标签搜索用 `tags.keyword` |
| `address` | `keyword` | 精确匹配 |
| `address_normalized` | `keyword` | EVM 地址统一小写 |
| `logo` | `keyword` | `index: false`，不参与搜索 |
| `pairs` 内层 | `nested` | 独立索引的数组对象 |
| `holders` 内层 | `nested` | 同上 |
| `heat_score` | `object` | 子字段 m5/h1/h6/h24 |
| 数值字段 | `double` / `integer` / `long` | 价格、交易量、时间戳等 |

#### 地址归一化（address_normalized）

EVM 地址在以太坊上区分大小写（EIP-55 checksum），但 Solana 的 Base58 编码本身大小写敏感。为了跨链统一查询，引入 `address_normalized` 字段：

```go
func NormalizeAddress(address string, chainID int) string {
    if IsEVMChain(chainID) {
        return strings.ToLower(address)
    }
    return address
}
```

查询时如果用 `address` 匹配不到，则回退到 `address_normalized` 查询。

#### 写入策略：全量索引 vs 部分更新

**全量索引（Mint）**：新代币收录时使用 `esClient.Index()` + `refresh=true`，确保写入后立即可搜索。此时会通过 struct embedding 注入 `address_normalized`：

```go
tokenWithNormalized := struct {
    model.Token
    AddressNormalized string `json:"address_normalized"`
}{Token: token, AddressNormalized: normalize(token.Address, token.ChainID)}
esClient.Index("web3_tokens", id, tokenWithNormalized)
```

**部分更新（Bulk）**：日常高频更新走 ES 的 `update` API + `doc` 合并，只传输有变更的字段：

```go
// es_writer.go: BWrite
bulkIndexer.Add(esutil.BulkIndexerItem{
    Action: "update",
    DocumentID: fmt.Sprintf("%s_%s", token.Network, token.Address),
    Body: token.MarshalEsUpdateJson(updateFields),  // {"doc":{"price_usd":0.5,"market_info":{...}}}
})
```

`MarshalEsUpdateJson` 根据 `updateFields` 列表，只序列化指定的字段到 `{"doc": {...}}` 结构中，减少网络传输量和 ES 的合并开销。

#### Bulk Indexer 配置

由 `go-elasticsearch` 官方 BulkIndexer 驱动：

```yaml
elasticsearch:
  worker_num: 8          # 并发 worker 数
  flush_bytes: 5242880   # 5MB 触发刷新
  flush_interval: 3      # 3 秒触发刷新
  write_timeout: 120
```

`update` 失败时（如 `document_missing_exception`），回退到 `IndexWithRefresh()` 创建文档。

## 三、核心技术实现

### 3.1 链抽象层

Solana 和 EVM 在底层有根本性差异，但业务层需要统一的操作接口。核心抽象如下：

```go
type Chain interface {
    FetchTokenMeta(ctx context.Context, token *model.Token) error
    FetchTokenInfo(ctx context.Context, token *model.Token) error
    FetchTokenSecurity(ctx context.Context, token *model.Token) error
    FetchTokenHolderInfo(ctx context.Context, network string, token *model.Token) error
    FetchTokenContractInfo(ctx context.Context, token *model.Token, mint event.MintEvent) error
    FetchTokenPiXiu(ctx context.Context, token *model.Token) error
}
```

这个接口定义了六大类操作，每个方法都接收 `*model.Token` 指针，直接在 token 对象上填充数据，而非返回新结构体——这是一种"传入填充"（fill-in）模式，减少内存分配和对象拷贝。

#### Solana 实现详解

Solana 的 RPC 接口基于 JSON-RPC 2.0，代币操作集中在几个核心方面：

**元数据读取**：Solana 上的代币元数据（名称、符号、图标 URI）存储在 Metaplex 的 Metadata 账户中。解析流程如下：

1. 根据 Mint 地址推导 Metadata PDA：`findProgramAddress([metadataPrefix, metadataProgramID, mintAddress])`
2. 通过 `getAccountInfo` 获取账户数据
3. 按照 Metaplex 的 TLV（Type-Length-Value）编码格式解码

核心逻辑是 PDA 推导——Solana 中每个代币的元数据地址不是随机生成的，而是通过程序派生确定：

```go
func DeriveMetadataPDA(mint solana.PublicKey) (solana.PublicKey, uint8) {
    seeds := [][]byte{
        []byte("metadata"),
        solana.MetadataProgramID.Bytes(),
        mint.Bytes(),
    }
    pda, bump, _ := solana.FindProgramAddress(seeds, solana.MetadataProgramID)
    return pda, bump
}
```

（实际代码使用 `solana.FindTokenMetadataAddress` 封装函数，功能等同上述逻辑）

Metaplex 的 `token_metadata` 包（`github.com/blocto/solana-go-sdk`）则用于反序列化元数据账户的原始数据Payload。两个 SDK 各有分工：gagliardetto 负责账号推导与 RPC 交互，blocto 负责 Metaplex 数据解析。

**持有者查询**：Solana 上的代币持有者通过 SPL Token 的 Token 账户模型表示。每个持有者有一个 TokenAccount，其中存储了 `mint`、`owner`、`amount` 字段。

查询策略需要考虑分页问题——热门代币可能有数万个持有者。分批查询的关键在于 `getProgramAccounts` 的 `dataSize` 和 `memcmp` 过滤参数：

```go
func (s *SolanaChain) FetchTokenHolderInfo(ctx context.Context, token string) (*HolderInfo, error) {
    mint := solana.MustPublicKeyFromBase58(token)
    accounts, err := s.rpc.GetProgramAccountsWithOpts(
        ctx,
        solana.TokenProgramID,
        &rpc.GetProgramAccountsOpts{
            Filters: []rpc.RPCFilter{
                {DataSize: 165},                       // TokenAccount 固定大小
                {Memcmp: &rpc.RPCFilterMemcmp{         // 过滤指定 Mint
                    Offset: 0,
                    Bytes:  mint.Bytes(),
                }},
            },
        },
    )
    // 解析每个 Account 的 data，提取 owner 和 amount
    // 过滤余额为 0 的账户
    // 计算分布统计
}
```

**内盘平台处理**：Solana 上有 Pump.fun、Moonshot 等"内盘"发射平台，这些平台有特殊的代币创建逻辑。例如某些平台的绑定曲线（Bonding Curve）在达到一定市值后会自动迁移到 Raydium——系统需要识别这种状态转换并更新代币的阶段标记。

#### EVM 实现详解

EVM 链利用 `go-ethereum` 库与链交互，核心差异在批量查询策略：

**Multicall3 批量查询**：EVM 链上单次 RPC 调用只能查询一个合约的一个方法。对于代币元数据（name、symbol、decimals、totalSupply），理论上需要 4 次 RPC。使用 Multicall3 合约可以在一次 RPC 中聚合多次合约调用：

```solidity
// Multicall3.aggregate 将多个 call 打包
struct Call {
    address target;    // 目标合约地址
    bytes callData;    // 编码后的函数调用
}

// 返回每个调用的结果
struct Result {
    bool success;
    bytes returnData;
}
```

Go 端的调用模式：

```go
func (e *EVMChain) metadataMulticall(ctx context.Context, token common.Address) (*TokenMeta, error) {
    calls := []multicall.Call{
        {Target: token, CallData: pack("name()")},
        {Target: token, CallData: pack("symbol()")},
        {Target: token, CallData: pack("decimals()")},
        {Target: token, CallData: pack("totalSupply()")},
    }
    _, results, err := e.multicall.Aggregate(calls)
    // 分别解码每个 result
}
```

**DEX 支持**：EVM 链上有多版本 DEX（Uniswap V2/V3/V4、PancakeSwap V2/V3/V4），每个版本的池子合约、事件格式、定价机制都不同。系统通过 The Graph 子图查询池子地址和流动性信息，并用一个 DEX 地址簿常量文件来识别池子所属的 DEX：

```go
var DEXAddresses = map[int64]map[string]DEXInfo{
    BSC: {
        PancakeV2Factory: {Name: "PancakeSwap", Version: "V2"},
        PancakeV3Factory: {Name: "PancakeSwap", Version: "V3"},
    },
    BASE: {
        UniswapV4Factory: {Name: "Uniswap", Version: "V4"},
    },
}
```

#### ChainClientManager

链客户端通过 Manager 以单例模式管理，Service 层只依赖 `Chain` 接口：

```go
type ChainClientManager struct {
    clients map[int64]Chain
    mu      sync.RWMutex
}

func (m *ChainClientManager) GetChain(chainID int64) (Chain, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    if client, ok := m.clients[chainID]; ok {
        return client, nil
    }
    return nil, fmt.Errorf("unsupported chain: %d", chainID)
}
```

在 Service 层，只需要根据事件中的 chainId 获取对应 Chain 实现即可：

```go
func (s *TokenService) enrichTokenMeta(ctx context.Context, chainID int64, token *model.Token) {
    chain, _ := s.chainManager.GetChain(chainID)
    _ = chain.FetchTokenMeta(ctx, token)   // 直接在 token 上填充数据
}
```

### 3.2 外部 API 客户端框架

系统对接了多个外部数据源，每个客户端共享一套基础架构：

```go
type Client struct {
    http    *http.Client
    limiter *rate.Limiter        // 令牌桶
    retry   retry.Option         // 指数退避
    metrics *prometheus.CounterVec
    circuit  *circuitBreaker     // 熔断器
}
```

**限速策略**：每个 API 客户端有独立的令牌桶，配置格式如 `"qps": 5, "burst": 10`。不同 API 的速率差异很大——CoinGecko Pro 允许 50 QPS，而某些免费 API 只有 1 QPS。

**熔断降级**：当一个数据源连续失败超过阈值（如 10 次），熔断器打开，后续请求直接返回缓存数据或跳过该源，避免级联故障。熔断器在半开后允许一个探测请求，成功则闭合。

**重试策略**：使用 avast/retry-go 实现指数退避：

```go
err := retry.Do(
    func() error { return doAPIRequest(ctx) },
    retry.Attempts(3),
    retry.Delay(500 * time.Millisecond),
    retry.MaxDelay(5 * time.Second),
    retry.DelayType(retry.BackOffDelay),
)
```

不同数据源在系统中的角色：

| 数据源                       | 提供数据                           | 重要性 |
| ------------------------- | ------------------------------ | --- |
| CoinGecko / CoinMarketCap | 美元价格、市值、交易量                    | 高   |
| GoPlus                    | 代币安全审计、合约权限                    | 高   |
| DexScreener               | DEX 价格、流动性、交易对                 | 高   |
| OKX Web3 / Binance Web3   | 代币图标、貔貅检测                      | 中   |
| Moralis                   | 持有者数据、NFT 关联                   | 中   |
| The Graph                 | 链上池数据子图查询(uniswapV4,pancakeV4) | 中   |
| Etherscan                 | 合约创建者及时间、合约源码、合约是否已验证          | 低   |
| Gate.io                   | 合约创建时间                         | 低   |

### 3.3 泛型 Kafka 消费者框架

三个 Kafka Consumer（Token、Trade、Metric）共享一个泛型框架，避免大量重复代码。

#### 消费者结构

```go
type Consumer[T any] struct {
    id          string
    workerSize  int
    buffers     []chan T
    logger      *zap.Logger
    kafkaReader *kafka.Reader
    limiter     *rate.Limiter
}
```

`MessageHandler[T]` 接口定义了每个消费者的处理逻辑：

```go
type MessageHandler[T any] interface {
    DispatchMessage(msg kafka.Message)
    HandleMessage(ctx context.Context, msg T)
}
```

#### 分片机制

这是保证数据一致性的关键设计。系统使用 CRC32 哈希将事件按 `network:tokenAddress` 键路由到固定的 buffer channel：

```go
func (c *Consumer[T]) hashBy(key string) int {
    hash := crc32.ChecksumIEEE([]byte(key))
    return int(hash) % c.workerSize
}
```

每个分片是一个独立的 channel + goroutine，串行处理路由到它的所有消息。这确保了同一个代币的多个事件不会被并发处理，避免竞态条件。

分片数在配置中设定（如 `worker_size: 16`），Worker 内部从 channel 中消费消息后调用 handler 的 `DispatchMessage` 方法进行实际分发。

#### 三种消费者的职责

**TradeConsumer**（主题：`trade_events`）——系统最核心的实时路径。注意 TradeConsumer 嵌入的是 `*Consumer[kafka.Message]`（原始消息），而非反序列化后的类型——这是因为 TradeHandler 有自己的 worker 池和批处理逻辑，泛型 Consumer 只负责路由：

```go
type TradeConsumer struct {
    *Consumer[kafka.Message]
    tradeHandler    *handler.TradeHandler
    tokenQueueCache *cache.TokenQueueCache
    repo            repository.Repository
}
```

TradeHandler 处理一条交易时的工作流：

```
Trade Event
  ↓ 
1. 反序列化 → EventTrade 结构
2. doTradeCache(trades) 批量处理
   ├── 更新代币市场数据（价格、交易量、流动性）
   ├── 更新持有者持仓（balance + 快照）
   ├── 检测聪明钱/开发者/狙击手标签
   ├── 检测内盘→外盘状态迁移
   ├── 排队池费查询
   └── 检测新代币 → 触发自动收录
```

**TokenConsumer**（主题：`token_events`）——处理代币生命周期事件：

- `mint`：新代币创建，触发收录流程
- `pool_migrate`：池子迁移（如从 Pump.fun 到 Raydium），更新池地址
- `supply`：供应量变更（铸币/销毁），更新供应数据
- `token_metadata`：元数据更新
- `token_platform`：平台标签更新

**MetricConsumer**（主题：`data_events`）——处理聚合行情数据：

- 价格和交易量更新
- FDV、市值重新计算
- 热度评分重新计算

#### 限速和背压

每个 Consumer 有令牌桶限速器，防止突发流量打死下游。当 Kafka 堆积时，限速器产生自然的背压（back pressure），Kafka consumer group 的 rebalance 机制会触发分区再均衡。

```go
func (c *Consumer[T]) processMessages(ctx context.Context, msgs []kafka.Message) {
    for _, msg := range msgs {
        c.limiter.Wait(ctx)                       // 令牌桶限速
        idx := c.hashBy(string(msg.Key))          // CRC32 分片
        c.buffers[idx] <- msg                     // 投递到对应 channel
    }
}
```

### 3.4 TokenUpdater — 缓冲写入器

高频写入是代币数据服务的核心瓶颈。TradeConsumer 每秒可能处理数千笔交易，每笔交易都可能触发代币数据的更新。如果逐条写入 Elasticsearch，ES 的写入瓶颈会很快暴露。

#### 设计原理

TokenUpdater 采用**分片 channel + Redis zset 队列 + 批量异步刷新**模式。与常见的"内存 map 合并后一次写入"不同，这里的缓冲分成两层：

- **L1：内存 channel 缓冲** — `tokenDataBuffer` 分片 channel，每个更新先进入对应分片，由 `upsertTokenDataRunner` 串行消费并合入 Redis
- **L2：Redis sorted set 队列** — 合并后的 key 写入 `token:need_flash` 和 `token:kafka_push` 两个 zset，autoFlash 从中批量读取

```go
type TokenUpdater struct {
    tl          *zap.Logger
    cfg         config.Config
    writers     map[string]BatchWriter[model.Token]
    kafkaWriter *TokenKafkaWriter
    tokenCache  tokencache.TokenCache
}
```

**合并策略**：同一个代币在 300ms 窗口内可能被多次更新。合并采用"后来者覆盖"原则，对于所有字段都是直接替换而非累加：

```go
// token_cache/cache.go: upsertTokenDataRunner
// 从内存或 Redis 中读取现有 token，调用 MergeToken 合并
mtoken.MergeToken(data.Token, data.UpdateField)
```

**MergeToken 的实现对所有字段都是覆盖写入**：

```go
// model/token_update_utils.go:155
func (t *Token) MergeToken(src Token, updateFields []string) {
    for _, f := range updateFields {
        switch f {
        case "market_info":
            if src.MarketInfo != nil {
                t.MarketInfo = src.MarketInfo  // 直接替换整个结构
            }
        case "price_usd":
            if src.PriceUSD != 0 {
                t.PriceUSD = src.PriceUSD
            }
        // ... 其他字段同理
        }
    }
}
```

注意 volume 类字段（M5Volume、H1Volume、H24Volume 等）是上游系统计算好的**绝对窗口值**，MetricEvent 中携带的数据已经是 5 分钟/1 小时/6 小时/24 小时各自时间窗口的成交量绝对值。因此合入时直接替换整个 MarketInfo 结构即可，不需要累加。

唯一有累加逻辑的地方是聪明钱追踪中的持仓统计：

```go
// service/token.go:932
token.SmartMoneyInfo.SmartMoneyTotalBuy += tradeInfo.Event.TxnValue
token.SmartMoneyInfo.SmartMoneyTotalSell += tradeInfo.Event.TxnValue
```

但这属于业务层面的持仓变动追踪，与 TokenUpdater 的合并无关。

**刷新流程**：autoFlash 使用 `for {}` 循环 + `time.Sleep`（而非 time.Ticker）：

```go
func (u *TokenUpdater) autoFlash() {
    for {
        tokenDatas, err := u.tokenCache.GetFlashToken()
        if err != nil { u.tl.Error(...) }
        if len(tokenDatas) == 0 {
            time.Sleep(300 * time.Millisecond)   // 空队列时等待
            continue
        }
        // 按 checksum 分 8 个并发分片写入
        for i := 0; i < 8; i++ {
            go func(id int) {
                u.flush(tokenMap[id], updateFieldsMap[id])
            }(i)
        }
        // 等待所有分片完成
        wg.Wait()
        time.Sleep(2 * time.Second)              // 每批次后冷却
    }
}
```

**Redis zset 队列**：关键区别在于缓存层不维护内存 map，而是将需要刷新的 key 写入 Redis sorted set：

```go
// upsertTokenDataRunner 每次合并完成后：
c.redisCache.ZAdd(ctx, TokenNeedFlashZsetKey, redis.Z{
    Score:  float64(time.Now().Unix()),
    Member: key,
})

// GetFlashToken 读取所有 key 并清空 zset：
func (c *TokenCacheImpl) GetFlashToken() (tokens []*TokenCacheData, err error) {
    c.tokenFlashLock.Lock()
    tokenKeys, _ := c.redisCache.ZRange(ctx, TokenNeedFlashZsetKey, 0, -1).Result()
    c.redisCache.Del(ctx, TokenNeedFlashZsetKey) // 清空队列
    c.tokenFlashLock.Unlock()
    return c.GetCacheDataByKeys(lo.Uniq(tokenKeys)) // 去重后获取数据
}
```

这种设计用了锁来保证 zset 的读取+删除是原子的，不像 map swap 需要读写锁。

**下游分发**：除了写入 ES，TokenUpdater 还需要将内盘状态变更推送到 Kafka：

```go
func (u *TokenUpdater) autoPushKafkaMsg() {
    for {
        tokenDatas, err := u.tokenCache.GetKafkaPushToken()
        // ... 类似 autoFlash，从 token:kafka_push zset 读取
        time.Sleep(500 * time.Millisecond)
    }
}
```

**错误处理**：当前实现中，flash 失败会记录错误日志和 Prometheus 指标，但**没有自动重试或回填机制**——失败后直接返回错误，不再重新入队。这意味着数据丢失风险依赖上游补偿队列兜底。

### 3.5 补偿队列

实时流处理无法保证 100% 的数据完整性。网络抖动、Kafka 堆积、RPC 超时都可能导致部分代币的数据缺失。

系统设计了三套基于 Redis 的自愈队列：

| 队列 | Redis Key | 轮询 | 数据结构 |
|------|-----------|------|----------|
| 安全审计 | `Web3_Token_Security_Audit_List` | 3s | Sorted Set (ZPopMax) |
| 数据检查 | `Web3_Token_Data_Check_List` | 5s | Sorted Set (ZPopMax) |
| 元数据缺失 | `Web3_Token_Miss_Metadata_List` | 30s/批 | List (LPop) |

安全审计和数据检查队列以固定的调度器间隔轮询，每次取出最高优先级的待处理项；元数据缺失队列则在每批处理完毕后 sleep 30 秒。

入队时机在实时处理的关键路径上：

```go
func (s *TokenService) tokenLoadWithOptions(ctx context.Context, token *model.Token) {
    chain := s.chainManager.GetChain(token.ChainID)
    // Chain 方法接收 *Token 指针，直接在原对象上填充
    _ = chain.FetchTokenMeta(ctx, token)
    _ = chain.FetchTokenInfo(ctx, token)
    err := chain.FetchTokenSecurity(ctx, token)
    if err != nil {
        s.addToSecurityQueue(token)   // 入补偿队列
    }
}
```

这种"实时流 + 补偿队列"的双轨设计是典型的**流批一体**思路：实时流保证正常情况下的低延迟，补偿队列兜底保证最终一致性。

### 3.6 热度评分系统

热度评分是一个综合指标，需要将多个维度的数据归一化到 0-100 分。

#### 评分模型

热度评分返回 `HeatScore{M5, H1, H6, H24}` 四个独立分数，每个分数在 [0, 100] 范围内：

```go
type HeatScore struct {
    M5  float64 `json:"m5"`
    H1  float64 `json:"h1"`
    H6  float64 `json:"h6"`
    H24 float64 `json:"h24"`
}
```

**计算流程**：

1. **基础分（所有窗口共享）**：`liquidityScore + holderScore`
   - `LogNormalize(TotalLiquidity) × 0.06`
   - `LogNormalize(HolderCount) × 0.04`

2. **各窗口独立分**：由四个分量相加
   - **交易量活跃度**：`LogNormalize(volume/divisor) × 0.4`（权重最高）
   - **交易笔数活跃度**：`LogNormalize(txns/divisor) × 0.2`
   - **价格动量**：`max(floor, LogNormalize(priceChange)) × 0.1`（floor 随窗口递增：0.2→0.4→0.6→0.8）
   - **代币年龄衰减**：`exp(-k·age)×0.1 + 0.1`（k 值随窗口递减，越老衰减越快）

不同窗口的除数不同：M5 ×2、H1 ÷4、H6 ÷25、H24 ÷100，使各窗口量级可比。

**早期退出条件**：
- 貔貅代币直接返回 basis `{0.1, 0.1, 0.1, 0.1}`
- 任何窗口价格跌幅 > 80% 时返回 basis
- 30 分钟无交易时返回 basis
- 交易量低于阈值时 `score ×= 0.1` 惩罚
- 5 分钟交易量为 0 时所有窗口 -10 分
- 某个窗口交易笔数为 0 时该窗口归零

**核心归一化函数**：

```go
func LogNormalize(value float64) float64 {
    result := math.Log(1+value) / (1 + math.Log(1+value))
    return result
}
```

曲线呈 S 型——低量时增长快，高量时趋于饱和，避免极端值主导评分。如果最终分数 < 1，则 `×= 100` 放大到可读区间。

**核心思想**：
1. **对数归一化 S 型曲线**：低量时增长快、高量时饱和，避免极端值主导评分
2. **多层防护**：价格崩盘 > 80%、30 分钟无交易、交易量为 0 等条件下直接压制评分，防止僵尸代币排名虚高
3. **流动性 + 持有者加分**：有真实流动性池和持有者基础的代币获得基础分加成

#### 排行榜维护

排行榜使用基于最小堆的 Top-N 数据结构：

```go
type Leaderboard struct {
    heap  *minHeap
    limit int
}

func (lb *Leaderboard) Add(score float64, item interface{}) {
    if lb.heap.Len() < lb.limit {
        heap.Push(lb.heap, &Entry{Score: score, Item: item})
    } else if score > lb.heap.Peek().Score {
        heap.Pop(lb.heap)
        heap.Push(lb.heap, &Entry{Score: score, Item: item})
    }
}

func (lb *Leaderboard) TopN() []interface{} {
    // 从堆中提取所有元素（已排序）
}
```

每个榜单类型（热门、新币、即将完成、已完成）各维护一个排行榜实例，在热度更新时同步更新。榜单数据最终写入 ES，通过 API 层暴露出去。

### 3.7 池费查询与缓存

DEX 流动性池的费率（Pool Fee）是计算交易成本的关键参数。不同 DEX、不同池子的费率不同（Uniswap V3 的费率等级有 0.01%、0.05%、0.30%、1.00% 等）。

**批量查询**：通过 Multicall3 批量查询多个池子的费率。系统维护一个内存中的池费缓存队列，TradeConsumer 在交易处理中发现新池子时，将池地址加入查询队列：

```go
type PoolFeeService struct {
    queue    chan common.Address
    cache    *FeeCache
    interval time.Duration  // 1s drain loop
}

func (s *PoolFeeService) drainLoop(ctx context.Context) {
    for range time.NewTicker(s.interval).C {
        addrs := s.drainQueue()
        if len(addrs) == 0 { continue }
        fees := s.batchQueryFees(ctx, addrs)
        s.cache.BatchSet(fees)
    }
}
```

**缓存策略**：池费数据使用 Redis + 本地内存两级缓存，TTL 设为 10 分钟。这是因为池费一旦设定极少变更（Uniswap V3 的池费在创建时就固定了）。

### 3.8 貔貅（蜜罐）检测

"貔貅"（又称蜜罐/Honeypot）是一种恶意代币，用户买入后无法卖出。系统使用专门的 Honeypot 检测 API（honeypot.is）进行判断：

```go
type HoneypotClient struct {
    httpClient *httpclient.Client
}

func (q *HoneypotClient) IsHoneypot(ctx context.Context, network string, address string) (IsHoneypotResponse, error) {
    var ret IsHoneypotResponse
    err := q.httpClient.Get(ctx, "https://api.honeypot.is/v2/IsHoneypot", ...)
    return ret, err
}
```

结果在 EVM 处理器的安全信息获取阶段使用：

```go
// evm.go: FetchTokenSecurity
isHoneypot, err := e.honeypotIs.IsHoneypot(ctx, token.Network, token.Address)
token.UpdateSecurityInfoByHoneypot(isHoneypot)
```

此外，GoPlus、OKX Web3、Binance Web3 等源也在安全检查中各自独立调用，每个源的检测结果都被独立合并到 SecurityInfo 中。但**没有集中的多源投票（voting）机制**——各源各司其职，最终 SecurityInfo 聚合了所有来源的结果。

### 3.9 聪明钱与开发者标签

系统维护了一套钱包标签体系，用于标记"聪明钱"、开发者钱包、做市商钱包等：

**聪明钱标签**：通过白名单机制——预先从 ES 索引 `web3_smart_wallets` 加载已知聪明钱包列表，使用 `CheckAddress(network, address)` 实时判断交易发起方是否属于聪明钱。如果是，则按以下规则标记：

```go
// token.go: UpdateExistingTokenFromTrade
if trade.Event.Side == "buy" {
    token.SmartMoneyInfo.SmartMoneyTotalBuy += trade.Event.TxnValue
    // 聪明钱净买入超过总供应量 5% 时打标签
    if token.SmartMoneyInfo.SmartMoneyTotalBuy -
       token.SmartMoneyInfo.SmartMoneyTotalSell > token.TotalSupply * 0.05 {
        token.Tags = append(token.Tags, utils.SmartMoneyBuyTag)
    }
} else if trade.Event.Side == "sell" {
    token.SmartMoneyInfo.SmartMoneyTotalSell += trade.Event.TxnValue
    // 聪明钱净卖出低于 5% 时移除标签
    if token.SmartMoneyInfo.SmartMoneyTotalBuy -
       token.SmartMoneyInfo.SmartMoneyTotalSell < token.TotalSupply * 0.05 {
        // 移除 SmartMoneyBuyTag
    }
}
```

**开发者标签**：代币创建者的钱包地址标记为开发者。如果开发者在代币创建后立即卖出，标记为"rug pull 风险"。

**狙击手标签**：检测在代币开盘后极短时间内买入的钱包——判断条件是交易时间戳与代币创建时间的差 < 10 秒，且不是创建者本人。符合条件的地址被标记为 [SniperBuy] 类型并记录在持有者信息中。

这些标签存储在 Redis 的 token 缓存中，代币信息中通过 tags 字段引用。

### 3.10 调度器设计

Scheduler 是一个进程内任务调度器，管理多个定时任务：

```go
type Scheduler struct {
    jobs       map[string]*ScheduledJob
    running    bool
    workerPool *pool.Pool
    logger     *zap.Logger
}

type ScheduledJob struct {
    name       string
    interval   time.Duration
    fn         JobFunc
    once       bool
    startDelay time.Duration
    maxRetries int
}
```

**启动顺序**：所有任务在 Core 初始化时注册完毕后统一启动。启动时按索引错开（**延时基于 index，不是随机 jitter**）：

```go
for i, job := range s.jobs {
    job.startDelay = time.Duration(i) * time.Second  // 第0个立即启动
    go s.runJob(ctx, job)
}
```

**定时机制**：使用标准的 `time.NewTicker`，每个 tick 触发一次执行，没有随机 jitter。

**超时控制**：每个任务通过 context.WithTimeout 控制执行时长，防止单个任务卡死协程。

**当前局限**：调度器没有分布式锁。在单实例部署时没问题，但多实例部署会导致任务重复执行。这是已知的架构待改进点。

## 四、数据流全景

### 4.1 实时事件流

```
Trade Event (Kafka)
    ↓
TradeConsumer.HandleMessage
    ↓ 批量解析，按 shard 分发
TradeHandler.doTradeCache(trades)
    │
    ├─▶ UpdateMarketData
    │     price,volume,liquidity,fdv → tokenCache
    │
    ├─▶ UpdateHolders
    │     balance snapshots → Redis → 分布统计
    │
    ├─▶ DetectSmartMoney
    │     检查发送方/接收方是否有聪明钱标签
    │     如果有，更新代币的聪明钱持仓
    │
    ├─▶ DetectSniper
    │     检查是否开盘狙击 → 更新狙击手标签
    │
    ├─▶ CheckInnerPoolStatus
    │     检测内盘是否已满 → 标记迁移
    │
    ├─▶ QueuePoolFeeQuery
    │     新池子 → 加入 PoolFeeService 队列
    │
    └─▶ CheckNewToken
         首次交易 → 触发 TokenLoadWithOptions
            ├─ FetchTokenMeta (Chain 接口)
            ├─ FetchTokenSecurity (Chain 接口)
            ├─ FetchTokenHolderInfo (Chain 接口)
            ├─ 外部 API 补全
            └─ SubmitMint → ES
    ↓
TokenUpdater.SubmitUpdate
    └─ TokenUpdater.autoFlash (300ms)
         └─ ES Bulk API
    └─ TokenUpdater.autoPushKafkaMsg (500ms)
         └─ Kafka inner_pool_updates
```

### 4.2 定时任务流

每个定时任务是一个独立的 Job，错峰启动后按固定间隔运行：

| 任务 | 间隔 | 功能 |
|------|------|------|
| Hotlist 补全 | 30s | 获取热门榜代币缺失数据 |
| Newlist 补全 | 30s | 获取新币榜代币缺失数据 |
| Toplist 发布 | 1min | 发布即将完成/已完成榜单 |
| DexScreener 增强 | 20min | 批量查询 DexScreener API 补全数据 |
| Pump/Goto 列表 | 10min | 更新内盘代币的列表状态 |
| 安全审计 | 3s(Queue) | 补偿队列轮询，执行安全检查 |
| 数据检查 | 5s(Queue) | 补偿队列轮询，补全缺失数据 |
| Metadata 补全 | 30s | 元数据缺失的重试 |
| Social 补全 | 5min | 社交链接信息更新 |
| Logo/Icon 补全 | 10min | 图标更新和高清覆盖 |
| Stablecoin 池 | 30min | 稳定币池数据更新 |
| 余额归零清理 | 1h | 清理零余额持有者 |

### 4.3 代币完整生命周期

```
1. 创建阶段
   链上 Mint 事件 → TokenConsumer
   → TokenLoadWithOptions()
   → 收录到 ES，初始状态：待补全

2. 早期交易阶段
   首笔 Trade 事件 → TradeConsumer
   → 更新市场数据，开始收录价格
   → 检测聪明钱和狙击手
   → 如果是内盘平台，进入内盘监控

3. 数据补全阶段 (Schedule Jobs)
   → 读取链上元数据 (Name/Symbol/Decimals)
   → 安全检查 (GoPlus/OKX/Binance)
   → 持有者统计
   → 社交信息 (Twitter/Discord/Telegram)
   → 合约信息 (创建者、源码)
   → 图标和品牌素材

4. 成熟阶段
   → 持续 Trade Event 更新
   → 热度评分计算和排行
   → 池费定期查询更新
   → 聪明钱信号追踪

5. 完成/下架阶段
   → 内盘代币达到条件后迁移
   → 标记生命周期结束
   → 发布完成榜
```

## 五、工程实践与设计模式

### 5.1 分层架构

```
Delivery:    Kafka Consumer → Handler
Business:    Service (x7)
Data Access: Repository (ES/PG/Redis) + Chain Interface (Solana/EVM)
Infra:       External API Clients + DB Connections + Cache
```

每层职责单一。Handler 负责消息解析和编排，Service 负责纯业务逻辑，Repository 和 Chain 负责数据访问。替换数据源只需实现对应接口，不影响上层。

### 5.2 配置驱动

几乎所有的行为都由配置控制：

```yaml
# config.worker.yaml (简化)
kafka:
  brokers: ["localhost:9092"]
  consumer:
    trade:
      topic: "trade_events"
      shards: 16
      rate_limit: 1000
    token:
      topic: "token_events"
      shards: 8
      rate_limit: 500

chains:
  solana:
    rpc_url: "https://api.mainnet-beta.solana.com"
    qps: 100
  bsc:
    rpc_url: "https://bsc-dataseed.binance.org"
    multicall: "0xcA11bde05977b3631167028862bE2a173976CA11"
    qps: 50
  base:
    rpc_url: "https://mainnet.base.org"
    multicall: "0xcA11bde05977b3631167028862bE2a173976CA11"
    qps: 50

apis:
  coingecko:
    base_url: "https://api.coingecko.com/api/v3"
    api_key: "${COINGECKO_API_KEY}"
    qps: 30
  goplus:
    base_url: "https://api.gopluslabs.io/api/v1"
    qps: 10
```

### 5.3 多层缓存

| 层次 | 技术 | 用途 | TTL |
|------|------|------|-----|
| L0 内存缓冲 | `chan T` + `memoryCache` | 分片 channel 缓冲 + token 内存缓存 | 1min |
| L1 本地缓存 | `sync.Map` + 自定义 Cache | Sniper 检测、聪明钱标签 | 1~5min |
| L2 分布式缓存 | Redis + go-redis | Token 数据、池费、余额 + zset 队列 | 24h |
| L3 持久存储 | ES | 完整的代币文档 | 永久 |

缓存的更新策略：
- L0 的 channel 由 `upsertTokenDataRunner` 消费；`memoryCache` 的 TTL 为 1 分钟
- L1 由对应的 Service 在写入后主动失效
- L2 的 Redis 通过 TTL 自动过期 + 读取时 lazy 刷新

### 5.4 监控指标

每个核心组件都暴露 Prometheus 指标：

| 指标 | 类型 | 标签 |
|------|------|------|
| `events_processed_total` | Counter | topic, handler |
| `events_processing_duration` | Histogram | handler |
| `token_updates_buffered` | Gauge | - |
| `es_bulk_write_bytes` | Histogram | - |
| `api_requests_total` | Counter | api, status |
| `api_request_duration` | Histogram | api |
| `consumer_lag` | Gauge | topic, partition |
| `jobs_duration` | Histogram | job_name |

### 5.5 日志与追踪

使用 zap 实现结构化日志，OpenTelemetry 实现分布式追踪：

```go
logger.Info("trade event processed",
    zap.String("network", event.Network),
    zap.String("token", event.TokenAddress),
    zap.Float64("amount", event.AmountUSD),
    zap.Int("batch_size", len(trades)),
    zap.Duration("elapsed", time.Since(start)),
)
```

每条日志包含 trace_id，关键路径上记录了处理耗时，方便定位性能瓶颈。

## 六、总结与思考

### 6.1 架构亮点

1. **链抽象层**将 Solana 和 EVM 的巨大差异封装在统一接口后，Service 层代码简洁且链无关。新增一条链只需要实现 Chain 接口，对业务逻辑零侵入。

2. **TokenUpdater 缓冲写入模式**在高吞吐场景下表现优异。300ms 窗口内的合并更新 + ES Bulk API 批量写入，将数万次/秒的写入请求合并为数十次/秒的 bulk 请求。

3. **补偿队列 + 实时流**的双轨设计，在保证正常情况低延迟的同时，通过补偿机制兜底保证最终一致性。这是典型的流批一体思想。

4. **泛型 Consumer 框架**让三个 Kafka Consumer 共享一套分片、限速、批量处理机制，大幅减少了重复代码。

5. **Redis zset 队列 + 内存缓存的双层缓冲**：合并后的更新存入 Redis sorted set 做持久化队列，autoFlash 消费时有原子性保证，重启不丢数据。

### 6.2 可改进之处

1. **定时任务无分布式锁**：当前假设单实例部署。如果扩展为多实例，需要用 Redis 分布式锁确保每个任务在全局只执行一次。

2. **PG 代币写入已禁用**：目前 ES 是代币的唯一持久存储，ES 故障会导致数据不可用。恢复 PG 的异步写入可以做跨存储冗余。

3. **外部 API 强依赖**：数十个外部数据源中的任何一个挂掉都会影响某些维度的数据完整性。需要系统化的降级策略文档和熔断阈值调优。

4. **"一次性"任务语义不统一**：某些被标记为 `RegisterOnceJob` 的任务实际上内部包含无限循环，命名和语义需要对齐。

5. **缺乏端到端测试**：当前主要依靠单元测试和日志监控，缺乏模拟 Kafka 输入 → 验证 ES 输出的集成测试。

### 6.3 适用场景

这套架构适合需要**实时处理多链代币数据**的中后台系统。如果你的业务场景也是从多条公链收集链上数据、聚合多源信息、并提供实时更新的数据服务——无论面向 DeFi、DApp 还是交易平台——这里面的链抽象、缓冲写入、补偿队列等设计思路都值得参考。

---

*本文基于实际生产项目经验整理，架构思路和代码片段仅供参考。*
