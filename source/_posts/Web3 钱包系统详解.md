---
title: Web3 钱包系统详解
date: 2025-05-06 00:10:00
tags:
  - Web3
  - 钱包
  - 托管钱包
  - 私钥
  - 多链
  - 交易系统
typora-root-url: ../
---

> 钱包不是一个按钮，也不是一个地址展示页。它是 Web3 产品里连接身份、资产、交易、安全和用户体验的核心系统。尤其在 Telegram Mini App、Trading Bot、Meme 预售和链上交易工具中，钱包系统往往同时包含托管钱包和端侧 Web3 钱包两种形态。

---

> **姊妹篇：[《从 Meme 预售到 Telegram Trading Bot：Web3 增长型交易产品全景入门》](/2025/05/05/从 Meme 预售到 Telegram Trading Bot：Web3 增长型交易产品全景入门)**——提供了该系列所分析的多个 Web3 增长型项目（预售站、Telegram Mini App 交易机器人、运营后台）的全景视角。本文是钱包系统的深入拆分篇，建议先阅读主线文档第 1-3 章建立全景认知。

---

## 1. 钱包系统到底负责什么

钱包系统至少负责五件事：

| 职责 | 说明 |
|------|------|
| 身份 | 用户是谁，是否有权限操作某个地址 |
| 控权 | 私钥在哪里，谁能签名 |
| 资产 | 用户有哪些链、哪些 Token、多少余额 |
| 交易 | 如何发起、签名、广播、追踪交易 |
| 安全 | 如何防止私钥泄露、误转、盗号和高风险交易 |

所以“钱包系统”不能只理解为 Web3 前端连接 MetaMask。真实产品里，钱包往往是一套跨前端、后端、数据库、第三方索引服务、链上 RPC、风控和用户认证的完整系统。

<!-- more -->

## 2. 两种钱包形态：托管钱包与端侧钱包

Web3 产品里常见两类钱包：

| 类型 | 私钥在哪里 | 用户体验 | 风险归属 |
|------|------------|----------|----------|
| 端侧 Web3 钱包 | 用户设备或钱包 App | 用户自己连接、签名、确认交易 | 用户主要负责私钥安全 |
| 托管/中心化钱包 | 平台生成、加密和保存 | 用户像使用 Web2 账户一样使用钱包 | 平台承担更高安全责任 |

这两种钱包经常同时存在。

例如，预售官网可能要求用户连接 MetaMask、OKX Wallet 或 WalletConnect 钱包进行购买和签名；而 Telegram Mini App 交易工具为了降低门槛，可能会给用户创建一个托管钱包，让用户直接在应用内查看资产、收款、转账和下单。

### 2.1 端侧 Web3 钱包

端侧钱包的核心特点是：平台不保存私钥，用户通过钱包软件签名。

典型流程：

```text
用户点击 Connect Wallet
  -> 选择钱包
  -> 钱包授权连接
  -> DApp 获取地址和链 ID
  -> DApp 请求签名
  -> 钱包弹窗确认
  -> DApp 拿到签名或交易哈希
```

常见能力：

- 连接钱包
- 监听地址变化
- 监听链切换
- 请求消息签名
- 请求交易签名
- 调用合约
- 查询余额
- 断开连接

常见技术：

- EVM：`wagmi`、`viem`、`ethers`、`RainbowKit`、`WalletConnect`
- Solana：`@solana/wallet-adapter-*`
- TON：`@tonconnect/ui-react`
- BTC：UniSat、OKX Wallet 等浏览器扩展或移动钱包能力

端侧钱包适合资产控制权要求高的场景，例如大额资产、外部用户钱包、公开预售购买、Claim 资格验证。

前端连接 EVM 钱包的典型代码（wagmi + viem）：

```tsx
import { useAccount, useConnect, useDisconnect } from 'wagmi';
import { useSignMessage } from 'wagmi';
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

function WalletConnector() {
  const { address, isConnected, chainId } = useAccount();
  const { connect, connectors } = useConnect();
  const { disconnect } = useDisconnect();
  const { signMessageAsync } = useSignMessage();

  // 连接钱包
  const handleConnect = async () => {
    await connect({ connector: connectors[0] });
  };

  // 消息签名
  const handleSign = async () => {
    const signature = await signMessageAsync({
      message: 'Login to MemePresale at ' + Date.now(),
    });
    // 将 address + signature 发送到后端验证
    await fetch('/api/auth/wallet', {
      method: 'POST',
      body: JSON.stringify({ address, signature, message }),
    });
  };

  // 读取链上数据
  const readBalance = async () => {
    const client = createPublicClient({ chain: mainnet, transport: http() });
    const balance = await client.getBalance({ address: address! });
    console.log('ETH balance:', balance);
  };

  if (!isConnected) return <button onClick={handleConnect}>Connect Wallet</button>;
  return <div>{address} <button onClick={() => disconnect()}>Disconnect</button></div>;
}
```

### 2.2 托管/中心化钱包

托管钱包的核心特点是：平台负责生成或保管私钥，用户通过平台账户使用钱包。

典型流程：

```text
用户进入应用
  -> 平台识别 Telegram / 邮箱 / JWT 身份
  -> 用户创建或激活钱包
  -> 后端生成私钥并加密保存
  -> 用户发起转账或交易
  -> 平台执行风控和二次验证
  -> 后端签名、广播、记录交易
```

托管钱包适合低门槛交易产品。用户不用理解助记词、RPC、链切换和签名弹窗，就可以开始使用资产和交易功能。

但托管钱包不是“简单模式”，而是“平台承担复杂度模式”。平台必须认真处理：

- 私钥生成是否安全
- 私钥是否加密保存
- 加密密钥如何管理
- 是否允许导出私钥
- 用户丢失 Telegram 或邮箱后如何恢复
- 高风险操作如何二次确认
- 平台内部人员是否能接触明文私钥
- 是否需要冷热钱包、限额、审计和告警

## 3. 钱包身份体系

钱包系统通常要同时处理 Web2 身份和 Web3 身份。

### 3.1 Web2 身份

Web2 身份用于识别应用用户，常见包括：

- Telegram 用户 ID
- Telegram init data
- 邮箱
- JWT
- 管理后台账号
- 邀请码或 UTM 来源

Telegram Mini App 中，Telegram 用户 ID 经常是核心用户主键。用户进入 Mini App 后，前端把 Telegram init data 传给后端，后端验证签名后建立登录态。

### 3.2 Web3 身份

Web3 身份用于证明用户控制某个链上地址。

常见方式是消息签名：

```text
后端生成待签名消息
  -> 前端请求钱包签名
  -> 用户确认
  -> 前端提交 signature
  -> 后端 recover 地址
  -> 判断 recovered address 是否等于用户声明地址
```

消息签名常用于：

- 登录
- 绑定外部钱包
- 生成 Claim 链接
- 转移历史订单资产
- 验证推荐关系

注意：消息签名只能证明“签名时控制这个地址”，不代表用户未来一直控制这个地址，也不代表他授权任意链上操作。

后端验证消息签名的典型实现：

```typescript
import { recoverAddress, hashMessage } from 'viem';

async function verifyWalletSignature(
  address: `0x${string}`,
  message: string,
  signature: `0x${string}`
): Promise<boolean> {
  try {
    const recovered = await recoverAddress({
      hash: hashMessage(message),
      signature,
    });
    return recovered.toLowerCase() === address.toLowerCase();
  } catch {
    return false;
  }
}

// 验证 Solana 签名
import { verifyMessageSignature } from '@solana/web3.js';
import { ed25519 } from '@noble/curves/ed25519';

function verifySolanaSignature(
  address: string,   // Base58 encoded public key
  message: Uint8Array,
  signature: Uint8Array
): boolean {
  const publicKeyBytes = bs58.decode(address);
  return ed25519.verify(signature, message, publicKeyBytes);
}

// 验证 TON 签名
import { signVerify } from '@ton/crypto';

async function verifyTonSignature(
  address: string,
  message: string,
  signature: Buffer
): Promise<boolean> {
  const publicKey = parseTonAddress(address);
  return signVerify(Buffer.from(message), signature, publicKey);
}

// 验证 BTC 消息签名
import { ECPair } from 'ecpair';
import * as bitcoin from 'bitcoinjs-lib';
import * as ecc from 'tiny-secp256k1';

bitcoin.initEccLib(ecc);

function verifyBtcSignature(
  address: string,
  message: string,
  base64Signature: string
): boolean {
  try {
    // Bitcoin Signed Message 格式:
    // magic = "\x18Bitcoin Signed Message:\n" + varint(len(message)) + message
    const msgBytes = Buffer.from(message, 'utf8');
    const magicPrefix = Buffer.from(
      `\x18Bitcoin Signed Message:\n${String.fromCharCode(msgBytes.length)}`
    );
    const hash = bitcoin.crypto.hash256(Buffer.concat([magicPrefix, msgBytes]));

    // 从签名中恢复公钥
    const sigBuffer = Buffer.from(base64Signature, 'base64');
    const recoveryId = sigBuffer[0] - 27 - 4; // uncompressed recovery
    const compressed = true;

    const publicKey = ecc.recover(hash, sigBuffer.subarray(1), recoveryId, compressed);
    const pubkeyBuffer = Buffer.from(publicKey);

    // 根据地址类型生成对应的地址进行比对
    // 支持 P2PKH (Legacy) 和 P2WPKH (Native SegWit)
    const networks = [bitcoin.networks.bitcoin, bitcoin.networks.testnet];
    for (const network of networks) {
      const p2pkh = bitcoin.payments.p2pkh({ pubkey: pubkeyBuffer, network });
      if (p2pkh.address === address) return true;

      const p2wpkh = bitcoin.payments.p2wpkh({ pubkey: pubkeyBuffer, network });
      if (p2wpkh.address === address) return true;
    }
    return false;
  } catch {
    return false;
  }
}

// BTC 签名验证的第二种方式——使用 UniSat SDK（更简洁）
import { address as btcAddress } from '@unisat/wallet-sdk';

function verifyBtcSignatureUniSat(
  address: string,
  message: string,
  signature: string
): boolean {
  try {
    // UniSat SDK 提供 decodeAddress 解析地址类型和网络
    const addrInfo = btcAddress.decodeAddress(address);
    // 消息按 Bitcoin Signed Message 格式哈希
    const msgHash = btcAddress.toMessageHash(message);
    // 恢复公钥并与地址比对
    const recoveredPubKey = btcAddress.recoverPubKey(msgHash, signature);
    const derivedAddress = btcAddress.publicKeyToAddress(
      recoveredPubKey, addrInfo.addressType, addrInfo.networkType
    );
    return derivedAddress === address;
  } catch {
    return false;
  }
}
```

### 3.3 身份绑定关系

一个真实系统里可能存在多种绑定：

| 关系 | 示例 |
|------|------|
| 用户 -> 托管钱包 | Telegram 用户拥有一个或多个平台钱包 |
| 用户 -> 外部钱包 | 用户绑定 MetaMask 地址用于 Claim |
| 钱包 -> Token 显示配置 | 某个地址隐藏或显示某些资产 |
| 用户 -> 推荐码 | 用户拥有 referralKey |
| 用户 -> 任务积分 | 用户完成活动并获得 points |

设计数据库时，需要明确“用户 ID”和“钱包地址”谁是主维度。交易型 Mini App 通常以用户 ID 为主；链上预售站则经常以钱包地址为主。

## 4. 私钥管理

钱包系统的最高风险点是私钥。

### 4.1 端侧钱包的私钥

端侧钱包中，私钥不进入平台系统。平台只拿到地址、签名和交易哈希。

平台要做的是：

- 不要求用户上传私钥
- 不在页面中收集助记词
- 不诱导用户授权危险合约
- 清楚区分消息签名和交易签名
- 对 Approve、Permit 等授权行为做解释

端侧钱包的风险更多来自钓鱼、恶意授权、假网站和用户误操作。

### 4.2 托管钱包的私钥

托管钱包中，平台必须有能力签名交易。常见方案包括：

| 方案 | 说明 |
|------|------|
| 后端加密保存私钥 | 实现简单，但平台责任最大 |
| KMS/HSM 管理密钥 | 私钥或加密密钥由专用密钥系统保护 |
| MPC / TSS | 私钥分片，多方协作签名 |
| 智能合约钱包 | 用合约账户实现恢复、限额、权限控制 |

如果是后端加密保存私钥，至少要考虑：

- 私钥生成使用安全随机源
- 明文私钥只在内存短暂存在
- 数据库只存密文
- 加密密钥不和数据库放在一起
- 日志绝不输出私钥
- 私钥导出必须二次验证
- 生产环境限制开发者直接访问密钥材料

后端加密保存私钥的代码实现（AES-256-GCM + Argon2 密钥派生）：

```typescript
import crypto from 'node:crypto';

const ALGO = 'aes-256-gcm';
const KEY_LENGTH = 32; // 256 bits
const IV_LENGTH = 16;  // 128 bits
const TAG_LENGTH = 16;
const SALT_LENGTH = 32;

// 从密码派生加密密钥
function deriveKey(password: string, salt: Buffer): Buffer {
  return crypto.pbkdf2Sync(password, salt, 100000, KEY_LENGTH, 'sha512');
}

// 加密私钥
function encryptPrivateKey(privateKey: string, passphrase: string): {
  encrypted: string;  // hex
  iv: string;         // hex
  salt: string;       // hex
  tag: string;        // hex
} {
  const iv = crypto.randomBytes(IV_LENGTH);
  const salt = crypto.randomBytes(SALT_LENGTH);
  const key = deriveKey(passphrase, salt);

  const cipher = crypto.createCipheriv(ALGO, key, iv);
  let encrypted = cipher.update(privateKey, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const tag = cipher.getAuthTag();

  return {
    encrypted,
    iv: iv.toString('hex'),
    salt: salt.toString('hex'),
    tag: tag.toString('hex'),
  };
}

// 解密私钥
function decryptPrivateKey(
  encrypted: string,
  passphrase: string,
  iv: string,
  salt: string,
  tag: string
): string {
  const key = deriveKey(passphrase, Buffer.from(salt, 'hex'));
  const decipher = crypto.createDecipheriv(
    ALGO, key, Buffer.from(iv, 'hex')
  );
  decipher.setAuthTag(Buffer.from(tag, 'hex'));
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

> **安全注意事项**：加密密钥（passphrase）不应硬编码在代码中或与密文存放在同一数据库。生产环境建议通过环境变量、密钥管理服务（AWS KMS / Vault）或 HSM 提供。

### 4.3 恢复与导出

托管钱包必须回答一个问题：用户如何恢复钱包？

常见选项：

- 不支持导出，只能在平台内使用
- 支持导出私钥，但需要 2FA
- 通过邮箱或 Telegram 恢复登录态
- 通过社交恢复或多因子恢复
- 使用 MPC 让用户持有一部分恢复材料

不同选择代表不同产品定位。如果钱包只是交易工具内部账户，可能不鼓励导出；如果承载真实资产，用户通常会要求能迁移资产控制权。

## 5. 多链钱包模型

多链钱包的难点是不同链的账户模型差异。

### 5.1 EVM

EVM 链包括 Ethereum、BSC、Base、Arbitrum、Polygon 等。它们共享相同的账户模型和地址格式，但链 ID、Gas 机制和 RPC 端点不同。

#### 地址与私钥

- 地址为 20 字节（0x + 40 位十六进制），示例 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
- 私钥为 32 字节 secp256k1 椭圆曲线密钥，与 Bitcoin 共用相同的曲线参数
- 同一私钥在所有 EVM 链上的地址相同——这对钱包系统既是便利（地址复用）也是风险（跨链事务混淆）
- BIP32/BIP44 派生路径：m/44'/60'/0'/0/0（Ethereum 的 coin type = 60）

#### 交易结构

一笔 EVM 交易包含这些字段：

| 字段 | 说明 | 关键点 |
|------|------|--------|
| nonce | 当前地址的交易序号 | 必须连续，跳 nonce 会导致交易卡住 |
| gasLimit | 最大 Gas 消耗量 | 简单转账 21000，合约交互根据复杂度估算 |
| maxFeePerGas | 单价上限（EIP-1559） | baseFee + priorityFee |
| maxPriorityFeePerGas | 小费 | 矿工/验证者优先打包的激励 |
| to | 目标地址 | 合约地址或空（部署合约） |
| value | 发送的 wei 数量 | 1 ETH = 10^18 wei |
| data | 合约调用 calldata | 函数选择器 + ABI 编码参数 |
| chainId | 链标识 | 防止重放攻击（EIP-155） |

#### Gas 机制

EIP-1559 之后的 Gas 计算：
```text
交易费用 = gasUsed x (baseFee + priorityFee)
baseFee：由网络拥堵自动调整，每个区块变动
priorityFee：用户设置的小费，决定交易优先级
```

Gas 估算与设置策略：
- eth_estimateGas：模拟执行估算 gasLimit
- eth_feeHistory：获取历史 baseFee 和 priorityFee 数据
- 常用策略：fast（高 priorityFee 抢跑）/ normal（标准优先级）/ slow（经济费率）
- 交易失败仍会消耗已使用的 Gas，无法退回

#### 合约交互

- ABI：定义了如何编码合约函数调用和解码返回值
- 函数选择器：keccak256 哈希的前 4 字节，标识调用的函数
- ERC-20 接口：balanceOf, transfer, approve, allowance
- EIP-712 Typed Data：结构化数据签名，用于 Permit（无 Gas 授权）、EIP-2612 等场景
- ERC-4337 Account Abstraction：UserOperation 替代传统 tx，实现社交恢复、Gas 代付等

#### 钱包集成实现

使用 viem 的典型实现：

```typescript
import { createWalletClient, createPublicClient, http, parseEther } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { mainnet } from 'viem/chains';

const account = privateKeyToAccount('0x...');

const walletClient = createWalletClient({ account, chain: mainnet, transport: http() });
const hash = await walletClient.sendTransaction({
  to: '0x...',
  value: parseEther('0.1'),
});

const publicClient = createPublicClient({ chain: mainnet, transport: http() });
const receipt = await publicClient.waitForTransactionReceipt({ hash });
console.log(receipt.status); // 'success' | 'reverted'
```

#### EVM 钱包的核心关注

EVM 钱包的复杂度不在于地址管理，而在于 Gas 策略、Approve 管理和跨链（chainId）处理。同一条私钥衍生地址可以用于所有 EVM 链，但每条链的资产和交易状态是独立的。

### 5.2 Solana

Solana 使用与 EVM 完全不同的账户模型。

#### 地址与密钥

- 地址为 32 字节的 Ed25519 公钥，Base58 编码：7EcDhSYGxXyscszYEp35KHN8vvw3svAuLKTzXwCFLtV
- 私钥通常为 64 字节的 Ed25519 密钥对（或 32 字节种子）
- BIP44 派生路径：m/44'/501'/0'/0'（Solana 的 coin type = 501）

#### 账户模型

Solana 没有 EVM 式的单合约状态——所有数据都存在独立的账户中：

| 概念 | 说明 |
|------|------|
| Account | 链上的存储单元，由 owner program 控制 |
| Program | 可执行账户（类似智能合约） |
| PDA（Program Derived Address） | 由 program 派生，没有对应私钥的地址 |
| ATA（Associated Token Account） | SPL Token 的标准持有账户，由用户地址和 Mint 地址派生 |

SPL Token 的余额查询与 EVM 的关键区别：
```text
EVM: balance = tokenContract.balanceOf(userAddress)
Solana: ata = findAssociatedTokenAddress(user, mint)
         balance = splToken.getAccount(connection, ata).amount
```

#### 交易结构

| 字段 | 说明 |
|------|------|
| message header | 签名者数量、只读账户数量 |
| accounts | 交易涉及的所有账户列表 |
| instructions | 程序调用指令列表 |
| blockhash | 交易有效期（约 60-120 秒），过期需重新签名 |
| compute units | 计算单元预算（默认 200K，可调） |
| priority fee | 优先费，以 microLamports/compute-unit 为单位 |

一笔交易可以包含多个 instruction，串行执行：

```typescript
import { Connection, PublicKey, Transaction, SystemProgram,
  sendAndConfirmTransaction, ComputeBudgetProgram } from '@solana/web3.js';

const connection = new Connection('https://api.mainnet-beta.solana.com');
const tx = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: sender.publicKey,
    toPubkey: recipient,
    lamports: 100000000,
  })
);
tx.add(ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 10000 }));

const { blockhash } = await connection.getLatestBlockhash();
tx.recentBlockhash = blockhash;
tx.feePayer = sender.publicKey;
const signature = await sendAndConfirmTransaction(connection, tx, [sender]);
```

#### 交易确认状态

Solana 的交易确认分为三级：

| 状态 | 说明 |
|------|------|
| processed | 被单节点确认（最快但有风险） |
| confirmed | 被集群确认（默认推荐） |
| finalized | 被集群最终确认（安全但较慢） |

#### Solana 钱包的核心关注

Solana 钱包的复杂度主要来自：ATA 需要预创建才能接收 SPL Token（需先支付创建费用）、Blockhash 短有效期导致移动端签名后广播容易超时、交易失败可能仍消耗费用、Compute Unit 估算不当导致交易失败。

### 5.3 TON

TON 的账户模型与前两者完全不同——钱包本身是一个智能合约，交易以异步消息传递。

#### 地址格式

TON 有两种地址格式：

| 格式 | 说明 | 示例 |
|------|------|------|
| raw（原始） | workchain + 32 字节哈希 | 0:83... |
| user-friendly（用户友好） | Base64 编码，含标志位 | EQD... |

用户友好地址包含 bounceable/non-bounceable 标志：
```text
EQD...  (bounceable：地址不存在时交易回弹)
UQD...  (non-bounceable：地址不存在时交易不回弹)
```

#### 钱包合约

TON 钱包本身就是部署在链上的合约，不是 EOA。有多个合约版本：

| 版本 | 特点 | 使用状态 |
|------|------|----------|
| v1/v2 | 早期版本，功能有限 | 已弃用 |
| v3/v3R1/v3R2 | 稳定版，seqno 验证 | 广泛使用 |
| v4/v4R1/v4R2 | 支持插件和订阅 | 当前主流推荐 |
| v5 | 最新版，改进插件机制 | 新项目推荐 |

#### 消息与交易

TON 没有 EVM 式的交易——所有状态变更通过消息（Message）传递：

```text
外部消息（External Message）
  -> 由用户发起，包含签名
  -> 钱包合约验证签名后执行
  -> 生成内部消息发送给其他合约

内部消息（Internal Message）
  -> 合约之间异步通信
  -> 不包含签名（调用者签名已在外部消息中验证）
```

一笔用户操作 = 一个外部消息（带签名）+ 钱包合约执行 + 零或多个内部消息。

#### 交易结构

| 字段 | 说明 |
|------|------|
| seqno | 钱包合约的交易序号，递增防重放 |
| valid_until | 消息过期时间（Unix 时间戳） |
| send_mode | 发送模式：0（普通）、128（携带全部余额）、64（携带剩余余额） |
| body | 消息体（Cell 结构，包含操作指令） |

#### 消息签名流程

TON 的签名发生在外部消息构建时，而非 EVM 式的独立签名请求：

```text
1. 构建 Cell 消息体（操作指令）
2. 构建外部消息结构（seqno, valid_until, body）
3. 将消息体哈希 -> 用 ed25519 私钥签名
4. 签名 + 公钥放入外部消息中
5. 将完整的外部消息发送到钱包合约地址
6. 钱包合约验证签名 && seqno 后执行
```

钱包集成实现：

```typescript
import { TonClient, WalletContractV4, internal } from '@ton/ton';
import { mnemonicToPrivateKey } from '@ton/crypto';
import { toNano } from '@ton/core';

const client = new TonClient({ endpoint: 'https://toncenter.com/api/v2/jsonRPC' });
const key = await mnemonicToPrivateKey(mnemonic.split(' '));
const wallet = WalletContractV4.create({ publicKey: key.publicKey, workchain: 0 });

const sender = client.open(wallet);
const seqno = await sender.getSeqno();

await sender.sendTransfer({
  seqno,
  secretKey: key.secretKey,
  messages: [internal({ to: 'EQD...', value: toNano('0.1') })],
});
```

#### TON 钱包的核心关注

TON 钱包的复杂度主要来自：钱包合约版本多样性（交易构建方法不同）、异步消息机制带来的交易确认不确定性（没有全局交易池）、Cell 序列化编码复杂度、bounceable/non-bounceable 地址选择。产品层必须警惕把 TON 简化成“另一个 EVM 链”。

### 5.4 Bitcoin 钱包

Bitcoin 是工程上与 EVM/Solana/TON 差异最大的链。集成 BTC 钱包必须理解 UTXO 模型、地址类型、手续费计算和 PSBT 签名机制。

#### 地址类型

Bitcoin 有四种主要地址格式，不同的钱包支持范围不同：

| 地址类型 | 前缀 | 说明 | 使用场景 |
|----------|------|------|----------|
| P2PKH (Legacy) | 1 | 最早期格式，交易体较大 | 矿池、老旧钱包 |
| P2SH-P2WPKH (Nested SegWit) | 3 | 兼容 Legacy 的 SegWit 地址 | 兼容场景 |
| P2WPKH (Native SegWit) | bc1q | 交易体小、手续费低，当前主流 | 现代钱包首选（如 UniSat） |
| P2TR (Taproot) | bc1p | 最新格式，支持 Schnorr 签名 | 高级场景（多签、隐私） |

实际项目中，Native SegWit（P2WPKH）是最常用的选择——它在手续费和兼容性之间取得了最佳平衡。

#### UTXO 模型

Bitcoin 没有“账户余额”的概念，余额是用户拥有的所有未花费交易输出（UTXO）的总和：

```text
UTXO 列表
  -> 每个 UTXO 包含：txid, vout, satoshis, scriptPk, addressType
  -> 转账时选择 UTXO，输入总和 >= 输出总和 + 手续费
  -> 多余部分作为找零返回给自己（需要找零地址）
  -> 交易体越大，手续费越高
```

工程上的关键点：
- UTXO 选择算法：优先选小额 UTXO 避免碎片，还是优先选大额 UTXO 减少交易体——取决于场景
- 找零管理：如果没有合适的找零 UTXO，下次交易会继续碎片化
- 粉尘标准：余额低于 546 sat 的 UTXO 被视为粉尘，不能单独花费

#### 手续费计算

Bitcoin 手续费不像 EVM 按 Gas 计费，而是按交易体大小（vsize，单位 vByte）和费率（sat/vB）计算：

```text
手续费 = 交易体大小 (vsize) × 费率 (sat/vB)
```

费率来自 Mempool 的推荐接口（getRecommendFee）：
- fastestFee：最快确认，适合紧急交易
- halfHourFee：半小时左右确认
- hourFee / economyFee：可接受等待

交易体大小取决于输入输出数量：1 个 P2WPKH 输入约 68 vB，1 个输出约 31 vB。输入越多，交易体越大，手续费越高。

#### 钱包实现模式

使用 bitcoinjs-lib + ecpair + tiny-secp256k1 的标准实现流程：

```typescript
const keyPair = ECPair.ECPair.fromWIF(wif, bitcoin.networks.bitcoin);
const { address } = bitcoin.payments.p2wpkh({
  pubkey: Buffer.from(keyPair.publicKey),
  network: bitcoin.networks.bitcoin,
});
// address -> bc1q...
```

PSBT（Partially Signed Bitcoin Transaction）是 BTC 钱包的标准签名方式。底层使用 `bitcoinjs-lib` 的签名方式：

```typescript
const psbt = new bitcoin.Psbt({ network: bitcoin.networks.bitcoin });

psbt.addInput({
  hash: utxo.txid,
  index: utxo.vout,
  witnessUtxo: { value: utxo.satoshis, script: Buffer.from(utxo.scriptPk, 'hex') },
});

psbt.addOutput({ address: recipient, value: sendAmount });
psbt.addOutput({ address: changeAddress, value: changeAmount });

psbt.signInput(0, keyPair);
psbt.finalizeAllInputs();
const txHex = psbt.extractTransaction().toHex(); // 广播用
```

实际项目中通常使用更高层的钱包 SDK（如 `@unisat/wallet-sdk`），它封装了 UTXO 选择、找零管理和 PSBT 构建：

```typescript
import { wallet, txHelpers, AddressType, NetworkType } from '@unisat/wallet-sdk';

// 1. 从 WIF 加载钱包
const deployerWallet = new wallet.LocalWallet(
  wifPrivateKey,
  AddressType.P2WPKH,       // Native SegWit
  NetworkType.MAINNET,       // 或 TESTNET
);

// 2. 通过 API 获取 UTXO
const btcUtxos = await unisatApi.getAddressUtxoData(deployerWallet.address);

// 3. 使用 SDK 构建 PSBT（自动选 UTXO + 找零）
const { psbt, toSignInputs } = await txHelpers.sendBTC({
  btcUtxos: btcUtxos.utxo.map(v => ({
    txid: v.txid,
    vout: v.vout,
    satoshis: v.satoshi,
    scriptPk: v.scriptPk,
    pubkey: deployerWallet.pubkey,
    addressType: deployerWallet.addressType,
    inscriptions: v.inscriptions,
    atomicals: [],
    rawtx: v.rawtx,
  })),
  tos: [{ address: recipientAddress, satoshis: sendAmount }],
  networkType: deployerWallet.networkType,
  changeAddress: deployerWallet.address,
  feeRate: fastestFee,      // sat/vB，来自 Mempool API
});

// 4. 签名 PSBT（自动 finalize）
const signedPsbt = await deployerWallet.signPsbt(psbt, {
  autoFinalized: true,
  toSignInputs,
});

// 5. 提取原始交易 hex
const rawTxHex = signedPsbt.extractTransaction().toHex();

// 6. 广播到 Bitcoin 网络
const txId = await unisatApi.pushtx(rawTxHex);
```

**关键签名 API** `deployerWallet.signPsbt(psbt, options)`：
- `psbt`：已构建但未签名的 PSBT 对象
- `autoFinalized: true`：签名后自动完成所有输入的 finalize
- `toSignInputs`：指定哪些输入需要签名（`[{ index, address }]`）

针对 inscription 转账的场景，SDK 提供专用的 `txHelpers.sendInscription()`，逻辑与 `sendBTC` 类似，但会确保包含 ordinal 的特定 UTXO 作为输入之一：

```typescript
const inscriptionInfo = await unisatApi.getInscriptionInfo(inscriptionId);

const { psbt, toSignInputs } = await txHelpers.sendInscription({
  assetUtxo: {               // 包含 inscription 的特定 UTXO
    txid: inscriptionInfo.utxo.txid,
    vout: inscriptionInfo.utxo.vout,
    satoshis: inscriptionInfo.utxo.satoshi,
    scriptPk: inscriptionInfo.utxo.scriptPk,
    pubkey: deployerWallet.pubkey,
    addressType: deployerWallet.addressType,
    inscriptions: inscriptionInfo.utxo.inscriptions,
  },
  btcUtxos: [...],           // 普通 BTC UTXO，用于支付手续费
  toAddress: claimAddress,   // 用户接收地址
  networkType: deployerWallet.networkType,
  changeAddress: deployerWallet.address,
  feeRate: fastestFee,
  outputValue: 546,          // 粉尘阈值
});

await deployerWallet.signPsbt(psbt, { autoFinalized: true, toSignInputs });
const rawTx = psbt.extractTransaction().toHex();
```

签名后的交易通过 `pushtx` API 广播，再通过 Mempool.space API 轮询确认状态：

```typescript
// 轮询确认
const pollTx = async (txId: string): Promise<boolean> => {
  while (true) {
    const status = await mempoolApi.getTransactionStatus(txId);
    if (status?.confirmed) return true;
    await new Promise(r => setTimeout(r, 3 * 60 * 1000)); // 3 分钟轮询
  }
};
```

#### 私钥格式

Bitcoin 私钥有多种格式：

| 格式 | 说明 | 示例 |
|------|------|------|
| WIF（Wallet Import Format） | 最常见的私钥格式，Base58 编码 | L5BcP... |
| 十六进制私钥 | 32 字节原始私钥 | 0x... |
| BIP39 助记词 | 12/24 个单词 | abandon ... |
| BIP32 扩展私钥（xprv） | HD 钱包根密钥 | xprv9s21... |

项目中通常使用 WIF 格式存放在环境变量中，通过 ECPair.fromWIF() 加载。

### 5.5 TRON

TRON（波场）虽然是 EVM 兼容链，但在钱包集成中有几个关键差异：

特点：

- 地址以 `T` 开头，Base58 编码，与 EVM 地址格式不同
- TRC-20 代币余额需要通过 TRON 节点的 `/wallet/triggerconstantcontract` 接口查询，而非标准 `eth_call`
- TRC-10 代币是 TRON 的原生代币标准，不需要合约，余额直接从账户资源中读取
- 交易需要 Bandwidth 和 Energy 两种资源，而非仅消耗 TRX 作为 Gas
- 资源不足时会燃烧 TRX 补充，导致实际扣费高于预期
- TRON 的能量（Energy）机制与 EVM Gas 机制不同：相同合约调用每次消耗能量相同（缓存），而 EVM Gas 按实际执行路径计算

钱包集成 TRON 时需要特别注意：地址格式转换（Base58 ↔ Hex）、资源估算而非 Gas 估算、TRC-10 与 TRC-20 两种代币模型的双重支持。

### 5.6 BRC-20 与 Ordinals

BRC-20 是 Bitcoin 生态中的一种实验性代币标准，基于 Ordinals 协议。它不是智能合约代币（如 ERC-20），而是通过 inscriptions（铭文）在 satoshi 上刻录 JSON 数据实现的。理解 BRC-20 对钱包系统意味着理解“代币状态在链下索引器维护”这一根本差异。

#### Ordinals / Inscriptions 基础

| 概念 | 说明 |
|------|------|
| satoshi | Bitcoin 的最小单位（1 BTC = 100,000,000 sat），每个 sat 可按发现顺序编号 |
| ordinal theory | 给每个 sat 分配一个序数（ordinal number），使其可被追踪和区分 |
| inscription | 在 sat 上刻录任意数据（图片、文本、JSON），使该 sat 成为“铭文” |
| BRC-20 | 在 inscription 中写入标准 JSON 格式来模拟代币操作 |

BRC-20 的关键工程差异：代币余额不由链上合约维护，而是由索引器解析所有 inscription 事件计算得出。钱包不能通过 RPC 调用查询 BRC-20 余额，必须依赖 UniSat API、OKX API 等第三方索引服务。

#### BRC-20 转账流程

BRC-20 转账不是简单的“构造交易、广播、确认”，而是涉及 inscription 创建的两步过程：

```text
Step 1: 创建 BRC-20 Transfer Inscription
  调用 UniSat Open API createBrc20Transfer(ticker, amount)
    -> 获得 orderId 和支付金额
  构造 PSBT 向 UniSat 支付 inscription 费用
  签名并广播
  等待 order 状态变为 minted

Step 2: 将 Inscription 发送给用户
  查询 inscription 的 UTXO 信息
  构造 sendInscription PSBT（将铭文 sat 发送到用户地址）
  签名并广播
  等待链上确认（mempool.space getTransactionStatus）
```

订单状态的完整周期：

```text
pending -> payment_notenough -> payment_overpay
       -> payment_withinscription -> payment_waitconfirmed
       -> payment_success -> ready -> inscribing -> minted
       -> closed | refunded | cancel
```

#### BRC-20 钱包依赖的外部服务

| 服务 | 用途 | API 示例 |
|------|------|----------|
| UniSat Open API | 创建 inscription 订单、查询 UTXO、推送交易 | POST /v2/inscribe/order/create/brc20-transfer |
| Mempool.space | 费率估算、交易确认状态、BTC 价格 | GET /api/v1/fees/recommended |
| OKX BRC-20 API | BRC-20 余额和交易记录 | 替代 UniSat 的索引器 |

#### 与 EVM 代币的关键差异

| 维度 | ERC-20 | BRC-20 |
|------|--------|--------|
| 状态存储 | 合约状态（链上可查询） | inscription JSON（需要索引器解析） |
| 余额查询 | balanceOf(address) RPC 调用 | 第三方 API 查询 |
| 转账 | 单笔合约调用交易 | 两步：创建 inscription -> 发送 inscription |
| 费用 | EVM Gas（可估算） | inscription 费 + BTC 网络费（波动大） |
| 确认时间 | ~12 秒（EVM） | ~10-60 分钟（BTC） |
| 精度 | 由合约 decimals 定义 | 固定为整数（索引器约定） |
| 安全性 | 合约逻辑执行 | 依赖索引器正确解析，无链上强制执行 |

#### BTC 钱包的工程模式总结

结合 bitcoinjs-lib 和外部 API 的 BTC 钱包典型分层：

```text
私钥层: WIF / mnemonic -> ECPair / HD wallet
地址层: publicKey -> p2wpkh / p2tr 地址
UTXO 层: UniSat API / 全节点 获取 UTXO 列表
交易层: PSBT 构建 -> 签名 -> 提取 hex
广播层: UniSat API pushtx / 全节点 sendrawtransaction
确认层: mempool.space getTransactionStatus / 全节点 gettransaction
```

这种模式不需要运行 Bitcoin 全节点，依赖外部 API 完成 UTXO 查询和交易广播，私钥控制在服务器端（托管钱包）或用户端（端侧钱包）。

## 6. 资产管理

资产页是钱包系统最重要的用户界面之一。

### 6.1 资产列表

资产列表通常由三部分组成：

```text
链配置
  + 钱包地址
  + Token 元信息
  + 链上余额
  + 价格
  + 用户显示配置
```

用户看到的是一行资产，但背后可能需要多个接口组合。

### 6.2 Token 元信息

Token 元信息包括：

- 合约地址
- 名称
- 符号
- 精度
- Logo
- 所属链
- 是否默认展示
- 是否用户自定义添加

元信息来源可能不稳定，所以本地数据库经常需要缓存和修正。

### 6.3 自定义 Token

自定义 Token 功能看似简单，但必须处理：

- 地址格式校验
- 链是否匹配
- Token 是否存在
- 精度是否正确
- 是否重复添加
- 是否有安全风险

错误的 decimals 会导致余额展示严重错误。

### 6.4 资产隐藏和显示

交易型钱包通常允许用户隐藏小额资产或垃圾 Token。

数据模型上需要保存：

- 用户 ID
- 钱包地址
- Token 地址
- 链
- 是否显示

这类配置是用户体验细节，但对资产页可用性影响很大。

## 7. 收款、转账与交易状态

### 7.1 收款

收款功能要解决的是“把正确地址给对的人”。

推荐展示：

- 当前链
- 当前钱包名称
- 地址
- 二维码
- 复制按钮
- 链错误提醒

多链场景中，应避免只展示一个地址却不提示链。尤其 EVM 地址跨链相同，用户容易误以为所有资产都能随便转。

### 7.2 转账

转账流程建议拆成几个阶段：

```text
选择资产
  -> 输入地址
  -> 校验地址
  -> 输入金额
  -> 检查余额
  -> 估算手续费
  -> 风险检查
  -> 用户确认
  -> 签名广播
  -> 状态追踪
  -> 完成或失败解释
```

每个阶段都应该有明确错误提示。不要把所有失败都显示为 “Something went wrong”。

### 7.3 交易状态

交易状态通常至少包括：

| 状态 | 说明 |
|------|------|
| Created | 本地订单或操作已创建 |
| Signing | 等待签名 |
| Submitted | 已广播到链 |
| Pending | 等待确认 |
| Success | 链上成功 |
| Failed | 链上失败 |
| Expired | 交易过期或超时 |
| Cancelled | 用户取消 |

如果系统有业务订单，还需要区分“链上交易成功”和“业务订单完成”。例如链上付款成功后，后端还要解析并写入订单。

## 8. Swap、限价单与 Snipe

钱包系统一旦进入交易场景，就不只是转账。

### 8.1 Swap

Swap 涉及：

- Token 选择
- 报价
- 路由
- 滑点
- Approve
- Gas
- 交易确认
- 失败回滚说明

用户最关心的是“我付出多少、收到多少、最少能收到多少、手续费多少、失败会不会扣钱”。

### 8.2 限价单

限价单涉及：

- 触发价格
- 订单过期时间
- 授权额度
- 取消订单
- 状态同步

限价单往往不是纯钱包能力，而是钱包、订单系统和交易执行服务的组合。

### 8.3 Snipe

Snipe 功能面向高风险高波动交易。它常见参数包括：

- 买入 Token
- 支付 Token
- 买入金额
- 最大滑点
- Gas 策略
- 超时时间
- 目标网络

Snipe 对风控要求更高，因为新 Token 常常存在流动性不足、合约后门、交易税、黑名单和 Honeypot 风险。

## 9. 钱包安全体系

钱包安全应该分层设计。

### 9.1 登录安全

常见机制：

- Telegram init data 校验
- JWT 过期时间
- 邮箱验证码
- TOTP 2FA
- 设备/IP 异常检测
- 退出登录

登录安全保护的是“谁能进入账户”。

### 9.2 操作安全

高风险操作应二次确认：

- 创建钱包
- 导出私钥
- 大额转账
- 添加陌生收款地址
- 执行高风险 Token 交易
- 关闭 2FA
- 修改邮箱

操作安全保护的是“进入账户后能做什么”。

### 9.3 交易安全

交易安全包括：

- 地址白名单
- Token 白名单
- Honeypot 检测
- 税费检测
- 合约权限检测
- 流动性检测
- 滑点限制
- 单笔限额
- 每日限额

交易安全保护的是“资产会不会因为交易设计缺陷或恶意合约损失”。

### 9.4 平台安全

托管钱包还必须考虑平台内部风险：

- 数据库泄露
- 日志泄露
- 环境变量泄露
- 运维误操作
- 后端接口被刷
- 内部人员越权
- 签名服务被滥用

如果平台能代用户签名，就必须把签名能力当成最敏感权限管理。

## 10. Telegram Mini App 环境下的钱包实现

Telegram Mini App 是托管钱包最常见的运行环境之一。它的 WebView 沙箱、无浏览器扩展、移动端交互限制，决定了钱包实现方式与桌面 DApp 有显著差异。

### 10.1 initData 身份绑定

Telegram Mini App 不依赖 MetaMask 等浏览器扩展，身份锚点是 Telegram 自身的 initData：

```text
用户打开 Mini App
  -> Telegram 客户端注入 initData（含用户 ID、auth_date、hash 等）
  -> 前端将 initData 发送到后端
  -> 后端使用 Bot Token 验证 HMAC-SHA256 签名
  -> 验证通过后建立会话（JWT / session token）
  -> 用户身份与钱包绑定
```

initData 的 hash 验证逻辑：

```typescript
import crypto from 'node:crypto';

function verifyTelegramWebAppData(initData: string, botToken: string): boolean {
  const urlParams = new URLSearchParams(initData);
  const hash = urlParams.get('hash');
  urlParams.delete('hash');

  // 按 key 排序，拼接 key=value
  const dataCheckString = [...urlParams.entries()]
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([k, v]) => `${k}=${v}`)
    .join('\n');

  // HMAC-SHA256 签名对比
  const secretKey = crypto.createHmac('sha256', 'WebAppData').update(botToken).digest();
  const computedHash = crypto.createHmac('sha256', secretKey)
    .update(dataCheckString)
    .digest('hex');

  return computedHash === hash;
}
```

### 10.2 WebView 生命周期管理

Mini App 在 Telegram 客户端内以 WebView 运行，生命周期与普通网页不同：

- `Telegram.WebApp.ready()`：通知 Telegram 页面已准备好，显示主界面
- `Telegram.WebApp.expand()`：展开 Mini App 到全屏
- `Telegram.WebApp.disableVerticalSwipes()`：禁用垂直滑动返回（交易页面建议启用）
- `Telegram.WebApp.onEvent('viewportChanged', handler)`：监听视口变化（键盘弹出/收起）
- 页面关闭时触发 `Telegram.WebApp.onEvent('themeChanged', handler)`，可在此保存未完成的交易草稿

钱包相关的关键注意点：

| 场景 | 处理方式 |
|------|----------|
| 签名弹窗时用户切出 | 监听 `visibilitychange`，超过一定时间自动取消签名请求并回到安全状态 |
| 交易提交后关闭页面 | 交易通过后端异步确认，页面重新打开时轮询状态 |
| 键盘弹出遮挡输入 | 使用 `viewportChanged` 事件调整布局，避免确认按钮被遮挡 |

### 10.3 托管钱包 Pin 码验证流程

Telegram 环境下无法使用浏览器扩展签名，托管钱包的"用户确认"通过 Pin 码实现：

```text
用户发起转账
  -> 前端弹 Pin 码输入面板
  -> 用户输入 6 位 Pin
  -> 前端将 Pin 哈希后发往后端
  -> 后端比对 hashed_pin == wallet.pinHash
  -> 验证通过后解密私钥、签名、广播交易
  -> 返回交易哈希
  -> 前端展示链上确认状态
```

Pin 码本身是用户体验和安全之间的平衡点。实际工程中需要额外考虑：

- Pin 码尝试次数限制（3-5 次失败后锁定）
- Pin 码重置需要 Telegram 身份二次验证（initData 重新校验）
- Pin 码输入面板使用 Telegram 的安全键盘模式（`Telegram.WebApp.showPopup`）
- 敏感操作（大额转账、导出私钥）建议叠加邮箱验证码或 2FA

### 10.4 移动端安全考虑

| 问题 | 影响 | 缓解方案 |
|------|------|----------|
| 截屏/录屏 | 地址、金额、私钥片段可能被截取 | `Telegram.WebApp.disableVerticalSwipes()` + 检测 `visibilitychange` 切到后台时模糊敏感信息 |
| WebView 注入 | XSS 可窃取 initData 和用户操作 | 不信任前端传来的任何数据；后端始终校验 initData 签名；不使用 `eval` 和内联脚本 |
| 键盘记录器 | Pin 码输入可能被记录 | 使用随机数字键盘布局而非系统键盘；输入后清除内存 |
| 剪贴板泄露 | 复制的地址可能被其他 App 读取 | 限制剪贴板写入提示；用户确认后再复制地址 |

### 10.5 LocalStorage 行为差异

Telegram Mini App 在 iOS 和 Android 客户端中，WebView 的 LocalStorage 行为不一致：

| 平台 | 问题 | 应对 |
|------|------|------|
| iOS | Mini App 关闭后 LocalStorage 可能被清除 | 关键数据（临时签名状态、未完成订单草稿）定期同步到后端 |
| Android | 长期使用时 LocalStorage 可能因空间不足被回收 | 缓存数据量控制在 1MB 以内；使用 `Telegram.WebApp.CloudStorage` 作为辅助持久化 |
| 两者 | 清除 Telegram 缓存会连带清除 Mini App 的 LocalStorage | 始终把用户身份和钱包绑定关系放在后端，前端仅缓存 UI 状态和 Token 元信息 |

### 10.6 移动端钱包 UX 设计要点

| 交互场景 | 设计建议 |
|----------|----------|
| 交易确认 | 签名请求时展示加载动画 + 链上确认进度条（已提交 → 已打包 → 已确认），成功/失败后震动反馈 |
| 地址输入 | 地址白名单快速选择 + 二次弹窗确认；支持扫码输入地址 |
| 数字输入 | 使用 Telelgram 安全数字键盘，提供 MAX 按钮快速填入全部余额，gas 调整滑条 |
| 多钱包视图 | 托管钱包与外部连接钱包分区域展示，避免用户混淆两类钱包的资产范围 |
| 错误提示 | 不要把失败显示为 “Something went wrong”——区分余额不足 / Gas 不足 / 链上失败 / 交易超时 / 用户取消，各自展示不同文案和解决建议 |

> **一条原则**：Telegram Mini App 的前端 LocalStorage 不可靠。所有用户身份、钱包绑定、交易状态的关键数据必须在后端持久化。

## 11. 数据模型设计

钱包系统的数据模型不是孤立的用户表 + 钱包表。它通常和订单、交易、积分、邀请、风控表紧密关联。以下基于 某 TG 交易工具的实际 Prisma schema，对比 多个预售项目的差异。

### 11.1 链配置（chain）

每条链的 RPC、浏览器、原生币信息必须集中管理，而非散落在代码中：

```prisma
model Chain {
  id          Int    @id @default(autoincrement())
  chainId     Int    @unique
  name        String // "ethereum", "bsc", "solana", "ton", "tron"
  symbol      String // "ETH", "BNB", "SOL", "TON", "TRX"
  decimals    Int    @default(18)
  rpcUrl      String
  explorerUrl String
  enabled     Boolean @default(true)
}
```

> **关键设计**：chain 字段在所有资产和交易表中必须存在。同一地址在 EVM 多链上资产完全不同——任何资产记录不带 chain 就是 bug。

### 11.2 用户与钱包

某 TG 交易工具的实际模型以 Telegram 用户为主维，支持多钱包：

```prisma
model User {
  id          Int      @id @default(autoincrement())
  tgId        BigInt   @unique
  username    String?
  firstName   String?
  lastName    String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  wallets     Wallet[]
  orders      Order[]
  points      PointReward[]
}

model Wallet {
  id                  Int      @id @default(autoincrement())
  userId              Int
  user                User     @relation(fields: [userId], references: [id])
  address             String
  chainGroup          String   // "evm", "solana", "ton"——同组链共享同一地址
  name                String?
  encryptedPrivateKey String?  // AES-256-GCM 密文，仅托管钱包存在
  salt                String?  // 密钥派生盐值
  pinHash             String?  // 交易 Pin 码哈希
  createdAt           DateTime @default(now())

  @@unique([userId, chainGroup]) // 每个用户每链组最多一个托管地址
}
```

> **链组（chainGroup）与地址复用**：EVM 多链（ETH、BSC、Base、Arbitrum）共享同一条椭圆曲线和派生路径，因此一个 `chainGroup = "evm"` 的钱包地址可接收所有 EVM 链的资产。存款时定向到同一地址，提现时按目标链选择地址。这减少了用户管理多地址的负担，但要求提现时必须指定目标链防止跨链误转。

### 11.3 交易与订单

交易模型需要同时记录链上状态和业务状态：

```prisma
model Transaction {
  id           Int      @id @default(autoincrement())
  userId       Int
  user         User     @relation(fields: [userId], references: [id])
  walletAddress String
  chainId      Int
  txHash       String?
  txType       String   // "buy", "sell", "transfer", "claim", "snipe", "stake", "unstake", "withdrawStake", "addLiquidity", "removeLiquidity", "contract"
  fromToken    String?
  toToken      String?
  fromAmount   Decimal?
  toAmount     Decimal?
  usdValue     Decimal?
  feeAmount    Decimal?
  feeToken     String?
  gas          BigInt?
  gasPrice     BigInt?
  nonce        Int?
  status       String   // "pending", "submitted", "confirmed", "failed", "expired"
  errorMessage String?
  createdAt    DateTime @default(now())
  confirmedAt  DateTime?
}
```

某 TG 交易工具使用 `orders` 表统一存储所有交易类型，通过 `tx_type` 枚举区分。而 多个预售项目 的预售场景则分离出独立的 `Order`、`StakeTransaction`、`Claim` 表，各自有专门的业务字段。

### 11.4 资产索引

```prisma
model TokenInfo {
  id        Int    @id @default(autoincrement())
  chainId   Int
  address   String
  symbol    String
  name      String
  decimals  Int
  logo      String?
  riskTags  String? // JSON 数组：["honeypot","selfdestruct","mintable","blacklist","highTax"]

  @@unique([chainId, address])
}

model TokenWallet {
  id            Int     @id @default(autoincrement())
  userId        Int
  walletAddress String
  chainId       Int
  tokenAddress  String
  isShow        Boolean @default(true)
  isCustom      Boolean @default(false)

  @@unique([userId, walletAddress, chainId, tokenAddress])
}
```

> **风险标签（riskTags）思路**：可以在 Token 信息中预留风险标签字段，用于存储社区标记或外部安全服务的检测结果。具体实现方式（如社区投票或自动化合约检测）需根据项目资源和安全要求决定。这里仅展示数据模型层面的设计思路。

### 11.5 业务相关表

某 TG 交易工具实现了一套完整的积分经济系统，与钱包、交易、推荐、质押和运营活动联动。积分发放时机包括：注册奖励、每日签到、推荐好友、完成运营任务、质押持仓。积分的消费场景包括：抵扣交易手续费、解锁钱包创建资格（消耗积分创建托管钱包）、解锁高级功能（Snipe 权限等）。

积分模型的核心设计点：

```prisma
model PointReward {
  id        Int      @id @default(autoincrement())
  userId    Int
  user      User     @relation(fields: [userId], references: [id])
  type      String   // "referral", "daily", "task", "staking", "signup"
  point     Int      // 正为增加，负为消费
  remark    String?
  createdAt DateTime @default(now())
}

model ActivityCode {
  id         Int       @id @default(autoincrement())
  code       String    @unique
  type       String    // "referral", "bonus", "event"
  pointValue Int       @default(0)
  maxUses    Int?
  usedCount  Int       @default(0)
  expiresAt  DateTime?
  createdAt  DateTime  @default(now())
}
```

### 11.6 风控相关表

```prisma
model TokenBlacklist {
  id          Int      @id @default(autoincrement())
  chainId     Int
  tokenAddress String
  reason      String?
  addedBy     String?
  createdAt   DateTime @default(now())

  @@unique([chainId, tokenAddress])
}

model WhitelistAddress {
  id        Int      @id @default(autoincrement())
  userId    Int
  address   String
  label     String?  // "exchange", "personal", "contract"
  createdAt DateTime @default(now())
}
```

### 11.7 跨项目数据模型对比

| 能力域 | 某 TG 交易工具 | 多个预售项目 |
|--------|-----------|-----------------|
| 订单模型 | `orders` 统一表，用 `tx_type` 区分 | 独立 `Order` / `StakeTransaction` / `Claim` 表 |
| 钱包模型 | `wallets` + `chain_group`，地址复用 | 无独立 wallets 表（依赖外部连接钱包） |
| Token 模型 | `token_infos` + `token_wallets` 完整索引 | 无，资产信息来自链上 + 第三方 |
| 用户 | `users` 表 + 多层关系（tgId 为主键） | `user_wallet` 或 tg id 为简单标识 |
| 风控 | `token_blacklist` + 风险标签 | 无独立风控模块 |
| 积分 | `point_rewards` + `activity_codes` | 简单 referral 记录 |

> **核心原则**：钱包表的 `chain` 字段不是可选项。任何资产记录（token 余额、交易历史、显示配置）不带链标识，在跨链场景中必然产生数据冲突。

## 12. 前后端职责分工

### 12.1 前端职责

前端负责：

- 展示钱包和资产
- 发起连接钱包
- 发起签名请求
- 输入金额和地址
- 展示 Gas、滑点、风险提示
- 展示交易状态
- 处理移动端体验

前端不应该负责：

- 信任前端计算的最终订单金额
- 保存明文私钥
- 决定交易是否真实成功
- 绕过后端风控

### 12.2 后端职责

后端负责：

- 验证登录态
- 验证签名
- 查询和缓存链上数据
- 校验交易结果
- 保存业务状态
- 执行托管钱包签名
- 风控检查
- 对接外部行情和索引 API

后端也不应该盲目信任第三方 API。关键资产状态仍应尽量通过链上数据或多个来源校验。

## 13. 常见坑

### 13.1 把地址当用户

在预售站中，钱包地址可能就是用户身份；但在 Telegram Mini App 中，一个 Telegram 用户可能有多个钱包。不要混淆。

### 13.2 把交易哈希当成功

交易哈希只说明交易被提交或可查询，不代表业务成功。

### 13.3 忽略 decimals

Token 金额必须按 decimals 换算。错误精度会导致展示、转账、报价全错。

### 13.4 忽略链 ID

同一个地址在多条 EVM 链上都存在，但资产完全不同。任何资产记录都必须带 chain。

### 13.5 托管钱包没有风控

如果平台保存私钥，却没有 2FA、限额、白名单、审计和告警，就是把交易所级别风险藏在普通应用里。

### 13.6 对用户解释不清签名

消息签名、交易签名、Approve 是三种完全不同的风险。产品文案必须区分。

## 14. 一个交易型钱包的最小闭环

如果要从零理解一个交易型钱包，可以按这个闭环看：

```text
用户登录
  -> 创建或绑定钱包
  -> 展示资产
  -> 添加或隐藏 Token
  -> 收款
  -> 转账
  -> Swap
  -> 查看订单和交易历史
  -> 风险提示
  -> 安全设置
```

每一步都对应工程模块：

| 用户动作 | 工程模块 |
|----------|----------|
| 登录 | Telegram / JWT / Auth |
| 创建钱包 | Key generation / encryption |
| 展示资产 | RPC / Moralis / TokenInfo |
| 转账 | address validation / gas / sender |
| Swap | quote / route / approve / tx |
| 订单 | backend order service |
| 风险提示 | token security / whitelist |
| 安全设置 | email / 2FA / audit |

理解这张表，基本就能读懂大多数 Web3 钱包产品的代码结构。

## 15. 钱包系统的演进方向

钱包产品正在从"连接外部钱包"走向更丰富的账户体系：

- 托管钱包降低新用户门槛
- MPC 降低平台单点私钥风险
- 智能合约钱包提供权限、恢复和限额
- Account Abstraction 改善 Gas 和签名体验
- Telegram Mini App 把交易入口前置到社交流量里
- 链上发币平台把 Token 发行和流动性启动标准化

未来的钱包不只是"资产保险箱"，更像是"交易账户 + 身份账户 + 增长账户 + 风控终端"的组合。

## 16. 学习建议

建议按这个顺序深入：

1. 先理解端侧钱包：地址、签名、交易、Approve。
2. 再理解托管钱包：私钥生成、加密、恢复、风控。
3. 然后理解多链差异：EVM、Solana、TON、BTC。
4. 接着理解交易：Swap、限价单、Snipe、Gas、滑点。
5. 最后理解安全：2FA、白名单、Token Security、审计。

钱包系统越往深处看，越不是单纯的 Web3 SDK 问题，而是产品、工程、安全和运营的交叉系统。

---

> **回到全景视角**：本文是《从 Meme 预售到 Telegram Trading Bot：Web3 增长型交易产品全景入门》的深入拆分篇。建议结合主线文档阅读，从产品全景到钱包工程细节建立完整认知。

## 参考资料

### 标准与协议
- [EIP-1193: Ethereum Provider API](https://eips.ethereum.org/EIPS/eip-1193)
- [EIP-191: Signed Data Standard](https://eips.ethereum.org/EIPS/eip-191)
- [EIP-712: Typed Data Signing](https://eips.ethereum.org/EIPS/eip-712)
- [EIP-4337: Account Abstraction](https://eips.ethereum.org/EIPS/eip-4337)
- [CAIP-2: Chain ID 标准](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md)
- [CAIP-10: Account ID 标准](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md)
- [CAIP-217: Asset ID 标准](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-217.md)

### 开发框架
- [wagmi: React Hooks for EVM](https://wagmi.sh/)
- [viem: TypeScript Interface for EVM](https://viem.sh/)
- [RainbowKit: Wallet Connection UI](https://www.rainbowkit.com/)
- [WalletConnect: Mobile Wallet Protocol](https://walletconnect.com/)
- [Solana Wallet Adapter](https://github.com/anza-xyz/wallet-adapter)
- [TON Connect UI React](https://github.com/ton-connect/ui-react)
- [@tonconnect/ui-react](https://www.npmjs.com/package/@tonconnect/ui-react)
- [bitcoinjs-lib: Bitcoin Transaction Library](https://github.com/bitcoinjs/bitcoinjs-lib)
- [ecpair: ECDSA Key Pair Management](https://github.com/bitcoinjs/ecpair)
- [@unisat/wallet-sdk: UniSat Wallet SDK](https://www.npmjs.com/package/@unisat/wallet-sdk)
- [bitcoin-address-validation: BTC 地址校验库](https://github.com/ruigomeseu/bitcoin-address-validation)

### BTC / BRC-20
- [UniSat Open API 文档](https://docs.unisat.io/dev/unisat-open-api)
- [Mempool.space API](https://mempool.space/api)
- [Ordinals 协议文档](https://docs.ordinals.com/)
- [BRC-20 标准说明](https://domo-2.gitbook.io/brc-20-experiment)
- [Bitcoin 地址类型 BIP 文档](https://en.bitcoin.it/wiki/Address)

### 安全与风控
- [GoPlus Token Security API](https://docs.gopluslabs.io/)
- [Honeypot.is 检测工具](https://honeypot.is/)
- [TokenSniffer 合约检测](https://tokensniffer.com/)

### Telegram Mini App
- [Telegram Mini App 官方文档](https://core.telegram.org/bots/webapps)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Telegram Web App initData 验证](https://core.telegram.org/bots/webapps#validating-data-received-via-the-web-app)

