---
title: 从 Meme 预售到 Telegram Trading Bot：Web3 增长型交易产品全景入门
date: 2025-05-05 00:00:00
tags:
  - Web3
  - Meme
  - Telegram Mini App
  - Trading Bot
  - Token Launch
  - 钱包
typora-root-url: ../
---

> 这是一份面向产品与工程理解的 Web3 增长型交易产品入门文档。它不把 Web3 只理解为钱包连接或链上交易，而是从 Meme Token 的发行、增长、认领、质押、交易工具、运营后台与发币基础设施演进几个层面建立完整视角。

---

## 1. 产品形态总览

Meme Token 相关产品通常不是单一形态，而是一组围绕“发币、获客、资产沉淀、交易转化”的组合系统。

早期常见做法是：项目方搭建官网预售页，用户通过钱包或法币入口购买代币，后台记录订单和分配额度，等到代币上线后再通过 Claim 页面完成领取。这个模式的核心是“项目方先组织流量和订单，再把资产交付到链上”。

随着 Telegram Mini App、移动端钱包和链上交易工具的发展，这类产品逐渐从“单次预售页面”变成了“持续交易入口”。用户不只是买一次 Token，还会在同一个入口里查看资产、领取奖励、参与任务、邀请好友、质押、查看行情、发起 Swap 或 Snipe。

可以把这种产品拆成四层：

| 层级 | 作用 | 典型能力 |
|------|------|----------|
| 增长入口层 | 获取用户和传播 | 官网、Telegram Bot、邀请链接、活动码、UTM |
| 资产承接层 | 记录用户权益 | 订单、Claim、积分、质押、推荐奖励 |
| 钱包与交易层 | 让用户持有和流转资产 | 钱包、余额、收款、转账、Swap、限价单、Snipe |
| 运营管理层 | 配置活动和修正数据 | 活动后台、任务上下线、积分配置、用户管理 |

这种产品最重要的理解方式，不是先看某个页面，而是先看用户生命周期：

```text
流量进入
  -> 绑定身份
  -> 购买或获得权益
  -> 资产确认
  -> Claim / Staking / Referral
  -> 钱包沉淀
  -> 行情与交易
  -> 持续活动与复访
```

<!-- more -->

## 2. Web3 基础概念

Web3 产品里的很多复杂度来自“用户身份、资产状态、交易执行”不再完全由服务器决定。

### 2.1 地址、私钥与签名

钱包地址是用户在链上的账户标识。不同链的地址格式不同，例如 EVM 地址通常是 `0x` 开头，Solana 地址来自 ed25519 公钥，BTC 地址有 Legacy、SegWit、Taproot 等多种格式。

私钥是控制地址资产的关键。谁掌握私钥，谁就能签名发起交易。因此钱包系统的第一原则是：私钥不能被随意暴露，任何“帮用户保管私钥”的设计都必须被视为高风险系统。

签名有两类常见用途：

| 类型 | 用途 | 示例 |
|------|------|------|
| 消息签名 | 证明“我是这个地址的控制者” | 登录、生成 Claim 链接、绑定钱包 |
| 交易签名 | 授权链上状态改变 | 转账、Swap、Approve、Mint |

消息签名不一定上链，但可以被后端验证；交易签名会被广播到链上，最终产生交易哈希。

### 2.2 Token、Gas 与交易哈希

Token 是链上的资产合约或原生资产。ETH、SOL、BNB 是链原生资产；USDT、USDC、Meme Token 通常是合约资产。

Gas 是交易执行成本。EVM 链上常见概念包括 `gasLimit`、`gasPrice`、`maxFeePerGas`、`maxPriorityFeePerGas`；Solana 则更多涉及优先费、计算单元和账户租金。

交易哈希是交易提交后的唯一标识，但“有交易哈希”不代表交易已经成功。产品侧通常需要继续追踪：

- 是否被链接收
- 是否打包进区块
- 是否成功执行
- 是否发生 revert / failed
- 是否达到业务所需确认数

### 2.3 EVM、Solana、TON、BTC 的差异

不同链不能只用“地址 + 余额 + 转账”三件事粗暴统一。

| 链类型 | 账户模型 | 常见复杂点 |
|--------|----------|------------|
| EVM | Account 模型 | Approve、Nonce、Gas、合约调用、Revert |
| Solana | Account / Program 模型 | ATA、租金、指令组合、优先费、交易过期 |
| TON | Cell / Message 模型 | 消息结构、钱包合约版本、异步确认 |
| BTC | UTXO 模型 | 找零、手续费率、UTXO 选择、地址格式 |

多链钱包系统的难点不只是“支持更多链”，而是要把完全不同的账户模型包装成用户可以理解的一套资产体验。

## 3. 钱包系统概览

钱包是这类产品里最容易被低估的模块。它表面上是“显示资产、复制地址、发起转账”，实际承担的是身份、资产、权限、安全和交易执行的核心职责。

这里需要先区分两种钱包形态：

| 钱包形态 | 私钥归属 | 产品表现 | 适合场景 |
|----------|----------|----------|----------|
| 端侧 Web3 钱包 | 用户自己掌握 | MetaMask、WalletConnect、OKX Wallet 等 | 预售购买、签名验证、外部资产接入 |
| 托管/中心化钱包 | 平台生成或保管 | Telegram 内置钱包、账户体系下的钱包 | 低门槛交易、资产页、积分/任务驱动的闭环 |

很多 Telegram 交易产品说自己是 Web3 钱包，但实际核心往往是托管钱包。用户不直接管理助记词，而是通过 Telegram 身份、邮箱、验证码、2FA 等方式访问平台生成的钱包。它降低了门槛，也引入了平台安全责任。

> “完整钱包系统详解请见姊妹篇[《Web3 钱包系统详解》](/2025/05/06/Web3 钱包系统详解)。”


## 4. 预售、Claim 与 Staking

Meme 预售系统通常围绕“先收款，后交付”展开。

### 4.1 预售购买

预售购买链路通常包括：

```text
选择支付链和 Token
  -> 输入支付金额
  -> 按当前阶段价格计算可获得数量
  -> 用户发起链上支付
  -> 后端检查交易
  -> 写入订单
  -> 更新已售金额和阶段状态
```

这里的关键是交易校验。不能只相信前端传来的交易哈希，后端需要解析链上交易，确认：

- 付款地址
- 收款地址
- Token 类型
- 实际金额
- 交易是否成功
- 是否重复提交
- 是否属于当前支持的链

### 4.2 Claim

Claim 是把预售权益转换成可流通资产的过程。常见设计包括：

- 用户连接原购买钱包
- 后端计算可领取数量
- 用户输入目标地址
- 用户签名证明身份
- 后端生成 Claim 数据或签名
- 用户或平台发起领取交易
- 后端记录已领取状态

Claim 的安全重点是防止重复领取、防止别人冒领、防止错误地址损失资产。

### 4.3 Staking

Staking 在这类产品中既是资产玩法，也是留存工具。实际项目中有两种实现模式：

**链上质押模式**：用户将 Token 存入 Staking 合约，收益由合约按固定 APR 发放。退出需等待锁定期满，或支付罚金提前取出。

**平台积分质押模式**：用户质押 Token 后，后端按日发放积分奖励（而非链上 Token）。积分可兑换交易手续费减免、提升排行榜排名、解锁高级功能。

两种模式的关键区别：

| 维度 | 链上质押 | 平台积分质押 |
|------|----------|-------------|
| 资产控制 | Token 进入合约 | Token 进入平台钱包 |
| 收益形式 | 链上 Token（固定 APR） | 平台积分（动态值） |
| 退出机制 | 锁定期满或罚金退出 | 即时退出（积分清零） |
| 与运营联动 | 弱（纯链上规则） | 强（可与活动/推荐/任务联动） |
| 工程复杂度 | 合约 + 事件监听 | 后端记录 + 定时任务 |

某 TG 交易工具实际支持多个质押池，APR 从 20% 到 150% 不等，各有不同的锁仓周期和积分倍率。后端通过定时任务计算每日收益并写入 `point_rewards` 表。

产品上需要解释清楚：

- 选择哪种质押模式
- 收益如何计算（固定 APR 还是动态分配）
- 是否可以提前退出
- 是否有排行榜（积分排名驱动竞争）
- 是否和积分、邀请奖励联动

### 4.4 Referral

Referral 是 Meme 项目最常见的增长机制。它通常通过推荐码或链接建立关系：

```text
邀请人生成 referralKey
  -> 被邀请人进入链接
  -> 系统绑定 srcRefer
  -> 被邀请人购买、交易或完成任务
  -> 邀请人获得 Token、积分或返佣
```

Referral 的难点是反作弊和归因。例如同一用户多账号、重复绑定、Telegram 参数篡改、UTM 与 Referral 冲突等。

## 5. Trading Bot / Mini App 交易系统

当产品从预售页进化为交易工具时，核心关注点从"卖出项目 Token"变成"让用户持续发现和交易 Token"。

### 5.1 交易类型全景

基于实际代码枚举，交易系统至少支持 12 种操作：

| 类型 | 说明 | 签名方式 |
|------|------|----------|
| Buy | 市价买入 Token | ETH: eth_sendTransaction / Solana: 交易序列化 |
| Sell | 市价卖出 Token | 同上 |
| Transfer | 转账 | 链原生转账 |
| Claim | 领取预售/空投资产 | 消息签名或合约调用 |
| Snipe | 新池/新 Token 快速买入 | 高 Gas 优先交易 |
| Limit | 限价单买入 | 条件触发后广播 |
| Stake | 质押 | 调用 Staking 合约 |
| Unstake | 解除质押 | 调用 Staking 合约 |
| WithdrawStake | 提取已解除质押资产 | 调用 Staking 合约 |
| AddLiquidity | 添加流动性 | 调用 DEX 合约 |
| RemoveLiquidity | 移除流动性 | 调用 DEX 合约 |
| Contract | 通用合约交互 | 自定义 calldata |

不同交易类型的确认特征不同：EVM 链依赖块确认数（通常 1-12 个块），Solana 关注最终性（finalized > confirmed > processed），TON 使用异步消息需跟踪消息哈希。

### 5.2 交易流程：从用户点击到链上确认

一笔交易的完整生命周期：

```text
前端发起
  -> 报价计算（滑点、路由、Gas）
  -> 用户确认
  -> 签名（端侧钱包: 钱包弹窗 / 托管钱包: Pin 码验证）
  -> 后端校验（风控、余额、地址）
  -> 路由到执行层（托管钱包 API / 自签名 / 广播）
  -> 链上广播
  -> 轮询确认（RPC / WebSocket）
  -> 更新数据库状态
  -> 前端推送结果
```

滑点计算的核心逻辑：

```typescript
// 买入时：用户支付 buyAmount，获得 output，最少保证 minOutAmount
const minOutAmount = (quoteOutput * BigInt(10000 - slippageBps)) / BigInt(10000);

// 卖出时类似，保证用户至少收到按滑点折价后的金额
```

交易失败重试策略：交易超时或失败后，系统不会立即放弃。常见做法是提高 Gas 重新广播（EVM）或更新 blockhash 重新签名（Solana），最多重试 3 次后标记失败并通知用户。

### 5.3 Token 风控思路

Token 风控是一个容易被低估的复杂问题。以下只是一些基础思路，真正的生产级风控需要结合专业安全服务：

风险标签枚举的思路：

| 标签 | 含义 | 检测思路 |
|------|------|----------|
| honeypot | 只能买入不能卖出 | 模拟买卖交易 |
| selfdestruct | 合约可自毁 | 检查合约字节码 |
| mintable | 可无限增发 | 检查合约权限 |
| blacklist | 合约有黑名单功能 | 检查合约地址映射 |
| highTax | 买卖税费异常 | 模拟交易获取实际到账 |
| mutexSniping | 抢跑/夹击攻击风险 | 检查交易排序依赖 |
| lowLiquidity | 流动性过低 | 查询 DEX 池子深度 |

一个简单的 Security check 流程参考：

```text
用户输入 Token 地址
  -> 查询本地黑名单缓存
  -> 如果命中则直接拦截
  -> 如果未命中则执行简单安全检查（模拟交易、合约分析）
  -> 生成风险评分摘要
  -> 低风险：正常展示，带上风险标签
  -> 高风险：弹窗警示，用户需手动确认继续
  -> 极高风险：直接拦截交易
```

> **注意**：上述描述仅提供设计思路参考。实际项目中风控标签的来源可能是社区投票或第三方 API，具体实现方式各有不同。生产级 Token 安全需要结合专业审计和实时监控，远超本文讨论范围。

### 5.4 Snipe 与限价单的技术本质

Snipe 和限价单虽然都涉及"条件触发后交易"，但本质不同：

| 维度 | Snipe | Limit Order |
|------|-------|-------------|
| 触发条件 | 新池创建 / 新 Token 部署 | 价格到达目标值 |
| 执行敏感度 | 毫秒级，抢在别人之前 | 秒级，价格到达即可 |
| Gas 策略 | 高优先费确保打包 | 标准 Gas |
| 存储方式 | 内存监听 + 即时广播 | 数据库持久化 + 条件检查服务 |
| 技术实现 | WebSocket 监听链上事件 -> 构建交易 -> 签名 -> 广播 | 定时任务扫描价格 -> 达到目标 -> 构建交易 -> 广播 |
| 风险等级 | 极高（新 Token 可能 rug / honeypot） | 中等（目标价格验证 + 流动性风险） |

Snipe 本质上是**时机敏感的一次性广播**，而限价单是**条件触发的持久化订单**。

## 6. 增长与运营系统

Web3 增长型产品经常把“交易”和“任务系统”揉在一起。

常见运营模块包括：

- 活动列表
- 活动上下线
- 活动排序
- 任务类型
- 活动链接
- Logo 与描述
- 积分奖励
- 活动开始、结束、下线时间
- 活动码
- UTM 参数

Telegram Bot 侧常见入口包括：

- `/start`
- startapp 参数
- inline keyboard
- web_app 按钮
- referral 参数
- claim 参数
- invite 参数

这些入口的本质都是把 Telegram 身份、推广来源和业务行为关联起来。

## 7. 工程架构速读

项目都是 pnpm 全栈设计，但包的划分方式随产品类型不同而不同：

| 能力域 | 预售站项目 | TG 交易工具项目 | 运营后台项目 |
|--------|-------------------|--------------------------|------------------------|
| 后端 + DB | `@project/backend` | `@bot/backend` (NestJS) | `@project/backend` |
| 前端 | `@project/frontend` | `@bot/frontend` | `@project/frontend` |
| 合约 | `@project/contracts` | 无（调用外部 DEX） | 无 |
| Web3 | `@project/web3` | `@bot/web3` | 无 |
| UI 组件 | `@project/ui-design` | `@bot/ui-design` | `@project/ui-design` |
| API 封装 | `@project/api` | `@bot/api` | `@project/api` |
| 共享配置 | `@project/shared` | `@bot/shared` | `@project/shared` |

共同的应用层数据流：

```text
页面组件 (Next.js pages router + tailwind)
  -> model hooks（hox 全局状态管理）
  -> api / tRPC client
  -> backend router (tRPC v10)
  -> service / external api / prisma
  -> database (PostgreSQL) or chain (RPC / Moralis聚合数据源 / 托管钱包API)
```

前端技术栈统一为 Next.js pages router + tailwind + hox。后端差异较大：部分项目使用标准 tRPC 路由，另一些选择 NestJS 作为后端框架。合约方面，只有预售站（某些预售项目）包含 Solidity 合约目录；TG 交易工具通过 托管钱包 外部服务和链上 DEX 合约交互，不维护自有合约。

这种分层的好处是前端不直接理解所有链上细节，而是通过 model 和 api 层获取“业务可用”的数据， 另外项目方把资产敏感和风险嫁接到托管钱包服务商抗雷。

## 8. 从中心化预售到链上公平发射

### 8.1 提到项目代表的中心化预售模式

本文分析的项目（多个预售、交易工具与运营后台项目）本质上都围绕“中心化预售 + 链上交付”展开：

- **项目方控制**：价格阶段、订单确认、Claim 发放都由后台数据库管理
- **数据库为核心**：所有订单、分配额度、推荐关系都记录在 Postgres 中
- **链上只做交付**：预售阶段在链上收币，Claim 阶段在链上发币，中间的业务逻辑由后端控制

这种模式的优点：灵活定价、营销可控、多链支付、推荐裂变易配置。局限：用户需要信任项目方按规则交付。

### 8.2 去中心化发行基础设施对比

近年的趋势是链上发行平台兴起，减少了后台的人工分配：

| 维度 | pump.fun | Meteora DBC | Raydium LaunchLab |
|------|----------|-------------|-------------------|
| 发射门槛 | 极低，任何人都可创建 | 中等，需配置曲线参数 | 中等，需在 Raydium 生态内 |
| 定价机制 | Bonding Curve（固定公式） | Dynamic Balance Curve（可参数化） | Bonding Curve + AMM 迁移 |
| 流动性迁移 | 达 $85K 市值后自动迁移到 Raydium | 自定义迁移条件 | 达到条件后自动上架 Raydium AMM |
| 交易时机 | 创建即交易 | 创建即交易 | 创建即交易 |
| 项目方控制 | 极低 | 中等（可配置曲线） | 低 |
| 适合场景 | 纯 Meme 快速发射 | 项目方有初始流动性需求 | 希望直接进入 Raydium 生态 |

### 8.3 这组项目的工作流复盘

从项目的实际运营过程可以抽象出 Meme 增长项目的工作流：

```text
预售网站获取用户（多个预售项目）
  -> 多链收款 + 后台订单管理
  -> Claim 页面发放 Token
  -> Staking 留存（链上质押 / 积分质押）
  -> Telegram Mini App 交易工具（交易工具项目）
  -> 推荐裂变 + 积分任务 + 运营活动（运营后台项目）
```

每个环节都对应一组明确的工程模块。这套工作流不是可选的，而是过去两年 Meme 项目在实践中沉淀出的有效模式。

### 8.4 未来工程重点变迁

随着链上发射平台成熟，工程重点正在转移：

| 阶段 | 工程重点 |
|------|----------|
| 预售站时代 | 搭建订单系统、多链支付处理、Claim 逻辑 |
| TG Mini App 时代 | 托管钱包、Token 行情聚合、交易执行、风控 |
| 链上发射时代 | 监控新平台池子（pump.fun/Meteora/Raydium）、Snipe 策略、流动性预警 |

未来的重点会从“如何搭一个预售页”转向“如何围绕链上发行基础设施做发现、交易、风控、增长和钱包体验”。钱包系统仍是核心，但随着发射去中心化，托管钱包的权重可能降低，端侧钱包和智能钱包（ERC-4337）的权重上升。

## 参考资料

### 链上发射平台
- [Meteora Dynamic Bonding Curve 文档](https://docs.meteora.ag/overview/products/dbc)
- [Raydium LaunchLab Overview](https://docs.raydium.io/user-flows/launchlab-overview)
- [pump.fun public docs / bonding curve 资料索引](https://deepwiki.com/pump-fun/pump-public-docs/3.1-bonding-curve-mechanism)

### 标准与协议
- [EIP-1193: Ethereum Provider API](https://eips.ethereum.org/EIPS/eip-1193)
- [EIP-712: Typed Data Signing](https://eips.ethereum.org/EIPS/eip-712)
- [EIP-4337: Account Abstraction](https://eips.ethereum.org/EIPS/eip-4337)
- [CAIP-2: Chain ID 标准](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md)
- [CAIP-10: Account ID 标准](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md)

### 开发框架
- [wagmi: React Hooks for EVM](https://wagmi.sh/)
- [viem: TypeScript Interface for EVM](https://viem.sh/)
- [RainbowKit: Wallet Connection UI](https://www.rainbowkit.com/)
- [WalletConnect: Mobile Wallet Protocol](https://walletconnect.com/)
- [Solana Wallet Adapter](https://github.com/anza-xyz/wallet-adapter)

