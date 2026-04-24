# Web3 × Flutter 面试知识点汇总

> 用途：面向 **Web3 移动端 / 钱包类 / DApp 容器** 岗位，把「链上常识 + 工程落地 + Flutter 技术栈」串成可复习清单。  
> 说明：本机 **BitKeep 相关工程未在常用路径检索到**（品牌已演进为 **Bitget Wallet**）；以下结合你曾提到的 **`xbit-app-mobile-v2` 依赖栈**（如 `web3dart`、`reown_appkit`、`turnkey_sdk_flutter`、`solana` 等）与业界通用实践整理。  
> **更新时间**：2026-04

---

## 1. 岗位在问什么（能力模型）

- **链基础**：账户模型、交易结构、区块确认、nonce、gas、链 ID、分叉与重组风险。
- **密码学**：哈希（Keccak-256）、签名曲线（EVM 常用 secp256k1；Solana 常用 ed25519）、随机数与密钥派生。
- **与合约交互**：ABI 编码、`eth_call` vs `eth_sendRawTransaction`、事件日志、常见标准（ERC-20/721/1155/Permit）。
- **多链与签名**：EIP-155 重放保护、EIP-712 Typed Data、不同链的 RPC/地址格式差异。
- **钱包连接**：WalletConnect / Reown AppKit、deeplink、会话生命周期、链切换。
- **托管与嵌入式钱包**：MPC/TSS、Turnkey 类方案、导出/恢复、风控与合规边界（了解即可，细节以官方为准）。
- **Flutter 工程**：安全存储、WebView 与 DApp 注入、GraphQL/REST、状态管理、错误与可观测性。

---

## 2. EVM 核心概念（面试必背）

| 概念 | 一句话 |
|------|--------|
| **Account** | EOA（私钥控制）与 Contract（代码控制）；余额/状态存在状态树。 |
| **Transaction** | `nonce` + `gas` + `to/data/value/chainId` 等字段；签名后广播。 |
| **Nonce** | 同一地址的交易序号；乱序/重复会导致失败或被拒。 |
| **Gas / EIP-1559** | 执行计价；`maxFeePerGas` / `maxPriorityFeePerGas` + `base fee`（概念面试常问）。 |
| **ChainId** | 链身份；**EIP-155** 把 `chainId` 编入签名，防止跨链重放。 |
| **RPC** | 读链：`eth_call`、`eth_getBalance`…；写链：发已签名 raw tx。 |
| **Event log** | 合约 `emit`；链下索引器（Subgraph/自建）常靠 topic/data 解析。 |

- **官方参考**：以太坊账户与交易概念 `https://ethereum.org/zh/developers/docs/accounts/`  
- **EIP-155**：`https://eips.ethereum.org/EIPS/eip-155`  
- **EIP-1559**：`https://eips.ethereum.org/EIPS/eip-1559`

---

## 3. 如何与合约交互（读 / 写 两条路）

### 3.1 只读（不改状态）

- 走 **`eth_call`**：本地执行，不上链；适合 `view/pure` 或任意只读调用（仍可能消耗节点资源）。
- 需要：合约地址 + **ABI** + 方法名/参数 → **ABI 编码 calldata**。

### 3.2 写入（改状态）

1. 组装交易：`to`（合约地址）、`data`（编码后的函数调用）、`value`（若 payable）、`gas` 相关字段、`nonce`、`chainId`。  
2. **本地签名**（私钥在客户端或安全模块内）→ 得到 **raw signed tx**。  
3. **`eth_sendRawTransaction`** 广播 → 拿到 `txHash` → 轮询 `receipt` / 等确认。  
4. 处理失败：`revert` 原因、out of gas、nonce 冲突、滑点/价格变动导致 swap 失败等。

### 3.3 常见标准（举例）

- **ERC-20**：`transfer/approve/allowance`；`approve` **无限授权**是安全面试高频坑。  
- **ERC-721 / 1155**：NFT 与半同质化；元数据 URI、批量接口差异。  
- **Permit（EIP-2612）**：链下签名授权，减少 `approve` 交易次数（需理解签名域与过期时间）。

---

## 4. 「不同链如何签名」——面试按这条线答

### 4.1 EVM 系（ETH / BSC / Polygon / Arbitrum…）

- **同一套私钥**可在多条 EVM 链生成**相同地址**；差异在 **chainId、RPC、区块浏览器**。  
- **签名里必须带 chainId（EIP-155）**：否则理论上存在跨链重放风险（面试点）。  
- **消息签名**常见三类：  
  - **`personal_sign`**：人类可读消息 + 前缀（防误签交易哈希）。  
  - **`eth_sign`**（历史/争议）：能签 raw hash，钓鱼风险更高，很多钱包默认限制。  
  - **EIP-712 Typed Data**：结构化域（`domain` + `types` + `message`），对 **Permit、登录挑战、订单签名** 很常用。  
  - 规范：`https://eips.ethereum.org/EIPS/eip-191` `https://eips.ethereum.org/EIPS/eip-712`

### 4.2 非 EVM（以 Solana 为例）

- 账户模型与 EVM 不同：**Program + Account**；交易里是 **instruction** 列表。  
- 密码学常用 **ed25519**（与 EVM 的 secp256k1 不同）。  
- 交易常带 **recent blockhash** 防重放；fee payer、签名者角色与 EVM 也不完全一样。  
- Flutter 侧常见：`solana` SDK + RPC（读账户/发交易）。

### 4.3 跨链 / L2 补充（点到即可）

- **桥**：铸币/销毁、消息传递、确认数与回滚风险。  
- **L2**：Sequencer、finalize/挑战期（因方案而异）；**用户体验**与 **安全假设** 是追问方向。

---

## 5. Flutter 侧常见技术栈（对照 `xbit-app-mobile-v2` 依赖思路）

| 方向 | 常见选型 | 面试怎么说 |
|------|-----------|------------|
| **EVM 读写** | `web3dart` | Dart 侧构造 tx、ABI 编解码、发 RPC；注意 isolate 与大 JSON。 |
| **钱包连接** | `reown_appkit`（WalletConnect 生态演进） | session、链切换、deeplink、配对；理解「DApp 请求 → 钱包授权」模型。 |
| **嵌入式 / MPC** | `turnkey_sdk_flutter` | 私钥不落地终端的业务形态、恢复码/策略、与后端会话边界。 |
| **Solana** | `solana` | 另一套账户/交易模型；与 EVM 不可混谈「同一套 ABI」。 |
| **密码学基础** | `pointycastle` / `crypto` / `ed25519_edwards` | 理解「不要自己发明加密」；知道何时用现成库与审计过的实现。 |
| **安全存储** | `flutter_secure_storage` | Keychain/Keystore；敏感 material 不进日志。 |
| **后端协议** | `graphql_flutter` + `http` | 行情/业务与链上解耦；链上失败与业务状态机要能解释。 |
| **DApp / H5** | `webview_flutter` / `flutter_inappwebview` | 注入 provider、deeplink 回跳、Cookie/存储隔离、反钓鱼。 |

---

## 6. 与 JS 技术栈（ethers / viem）对照（面试加分）

| 维度 | JS（ethers/viem） | Flutter（常见） |
|------|-------------------|------------------|
| ABI/合约 | `Contract` + `Interface` | `web3dart` 的 `DeployedContract` / ABI 编解码 |
| Provider | JSON-RPC Provider | `Web3Client` + 自建 endpoint |
| Wallet | browser extension / sdk | 自建密钥库、或 WalletConnect、或 MPC SDK |
| 并发模型 | Promise/async | `Future`/`Stream`；重计算考虑 `compute`/Isolate |

---

## 7. 安全与风控（Web3 岗必问）

- **私钥与助记词**：永不打印、不进日志、不进崩溃上报；屏幕录制/剪贴板时效。  
- **钓鱼**：假站点 `personal_sign` 授权、恶意 `approve`、伪造 RPC。  
- **供应链**：依赖锁定、签名插件来源、内嵌脚本更新策略。  
- **交易预览**：展示「真实 spender / 真实 token / 真实额度」，避免盲签。  
- **合规提示（了解）**：不同地区对托管/KYC 要求不同，面试点到「产品/法务介入」即可。

---

## 8. 面试高频追问（短答）

- **为什么需要 chainId？** 防跨链重放；签名与交易格式要一致。  
- **为什么 approve 危险？** 授权第三方花你的 token；无限额度 + 恶意合约可掏空。  
- **为什么读链还要 ABI？** 知道如何把参数编码进 `data`、如何把返回值解码。  
- **为什么 Flutter Web3 也要懂 WebView？** 很多 DApp/H5、登录态、deeplink 回跳、注入与隔离。  
- **为什么需要索引器？** 链上日志分散；复杂查询（持仓、成交）通常走后端/Subgraph。

---

## 9. 官方与权威延伸阅读（防链接失效：以域名为准）

- Flutter 安全存储：`https://pub.dev/packages/flutter_secure_storage`  
- Ethereum JSON-RPC：`https://ethereum.org/zh/developers/docs/apis/json-rpc/`  
- WalletConnect / Reown：`https://docs.reown.com/`  
- EIP 仓库：`https://eips.ethereum.org/`

---

## 10. 维护建议（你后续要补 BitKeep/老项目时）

- 若能找到旧仓库：重点抽取 **「签名流程图 + RPC 配置 + 合约调用封装 + 多链切换」** 四页笔记即可，不必整仓搬进笔记站。  
- 与 `xbit-app-mobile-v2` 对齐时：优先记录 **业务状态机（链上失败 vs 业务单）**、**钱包会话生命周期**、**风控交互（预览/二次确认）**——这些比「会调某个 API」更像高级岗追问。
