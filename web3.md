# Web3

## 1) 协议总览（前端“为什么/怎么用/边界”）
**面向连接与 RPC**
- **EIP-1193 Provider**：统一钱包 ⇄ dApp 的接口与事件（`request`、`accountsChanged`、`chainChanged`）。  
  *前端用法*：以 provider 为**能力探测与事件源**，其上封装连接/断开/切换网络。
- **EIP-1474 JSON-RPC**：以太坊 RPC 方法与错误码标准化（`eth_call`、`eth_estimateGas`、`eth_sendRawTransaction`、`eth_getLogs`…）。  
  *前端用法*：读写链的**最低共同层**；在 viem/ethers 里最终都落到这些方法。
- **CAIP-2/10**：链与账户的通用标识（例：`eip155:1:0xabc…`）。  
  *前端用法*：WalletConnect v2 会话协商的**命名空间**与**权限**底座。

**面向交易与费用**
- **EIP-1559**：`baseFee` 随块自调，用户给 `maxFeePerGas` 与 `maxPriorityFeePerGas`。  
  *前端用法*：用 `eth_feeHistory` 估优先费；替换交易（加速/取消）要**同 nonce、更高费**。
- **EIP-3085/3326**：`wallet_addEthereumChain` / `wallet_switchEthereumChain`。  
  *前端用法*：引导钱包**添加/切换**网络；不支持时**降级提示**。
- **EIP-4844**（了解）：Blob 交易降低 L2 数据费。  
  *前端用法*：费用显示要包含 **L1 data** + **L2 exec**，对账展示“Blob 费”。

**面向签名与身份**
- **EIP-191 / `personal_sign`**：原始/文本签名（易被钓鱼）。  
- **EIP-712**：结构化数据签名（域分隔、防重放）。  
  *前端用法*：**授权/订单/登录**一律用 712，UI 明示 `domain.name/chainId/contract`。
- **EIP-1271**：合约账户签名校验接口（`isValidSignature`）。  
  *前端用法*：验证 Safe/AA 的签名。
- **EIP-4361 (SIWE)**：Web3 登录挑战-响应格式。  
  *前端用法*：前端生成 challenge，用户 712 签名，服务端校验与建会话。

**面向支付与代币**
- **ERC-20/721/1155**：代币/NFT 基础。  
- **EIP-2612（Permit）/Permit2**：离线授权，链上 `transferFrom` 代付。  
  *前端用法*：减少一次 on-chain `approve`，并**限制额度/到期**。
- **EIP-681**：支付 URI（`ethereum:…`）。  
  *前端用法*：收款二维码/链接的标准格式。

---

## 2) 多钱包连接：基于 EIP-1193 的状态机与事件
**目标**：统一 connect/ disconnect/ switch account & chain，避免“幽灵连接”。

**状态机（简化）**
```
Idle
 ├─ connect(provider) ─→ Connected{account, chainId, provider}
 │     ├─ accountsChanged[] ─→ Idle（断开）
 │     ├─ accountsChanged[a≠curr] ─→ Connected{new a}
 │     ├─ chainChanged ─→ Connected{…new chain}
 │     └─ disconnect/error ─→ Idle + toast
 └─ fail → Idle
```

**落地要点**
- **能力探测**：优先从连接器（Injected / WalletConnect / Coinbase SDK）创建 provider；不要直接假设 `window.ethereum` 存在（SSR/移动端）。
- **只读与可写分层**：读用独立公共 RPC（稳定/限流大），写用钱包 provider（带权限）。
- **持久化**：把“**选择的连接器**”持久化（localStorage），而**账号/网络**通过事件实时刷新（不持久化敏感地址）。

**示例（viem + wagmi 概念同等适用）**
```ts
import {createPublicClient, http, createWalletClient, custom} from 'viem';
const publicClient = createPublicClient({ chain, transport: http(PUBLIC_RPC) });

async function connectInjected() {
  if (typeof window === 'undefined' || !window.ethereum) throw new Error('No provider');
  const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' }) as string[];
  const walletClient = createWalletClient({ chain, transport: custom(window.ethereum) });
  window.ethereum.on('accountsChanged', onAccounts);
  window.ethereum.on('chainChanged', onChain);
  return { account: accounts[0], walletClient };
}
```

---

## 3) WalletConnect v2：Pairing vs Session、命名空间、续期/重连
**协议原理**
- **Pairing**：一次性的“设备握手通道”（QR/Deep Link 扫码得到 `wc:` URI），产生 **topicA**。  
- **Session**：在 Pairing 之上协商**权限**（命名空间 `eip155:{chains}`、方法、事件）→ 形成 **topicB**，dApp 后续用 Session topic 通信。  
- **CAIP-2/10**：会话内多链/多账号以 `eip155:1:0x…` 表达。

**前端实现点**
- **多链多账号**：在 `requiredNamespaces` 声明允许链与方法；会话建立后从 `namespaces` 中取出**已授予**的 (chain, accounts)。  
- **断线重连**：SignClient 重启后用 **已保存的 Session 列表**直接 `emit` 可用；失效则重新 pairing。  
- **续期**：临期前刷新或在 `session_delete` 事件后引导重新连接。

**最小伪代码**
```ts
const sign = await SignClient.init({ projectId, relayUrl: 'wss://relay.walletconnect.com' });

async function connectWC() {
  const { uri, approval } = await sign.connect({
    requiredNamespaces: { eip155: { chains: ['eip155:1','eip155:137'], methods: ['eth_sendTransaction','personal_sign'], events:['accountsChanged','chainChanged'] } }
  });
  if (uri) showQr(uri); // 或 mobile deep link
  const session = await approval();
  save(session); // 持久化
}

sign.on('session_update', e => update(e.params.namespaces));
sign.on('session_delete', () => logout());
```

---

## 4) 链切换：EIP-3085 vs EIP-3326 与降级策略
- **EIP-3326 `wallet_switchEthereumChain`**：在钱包**已有**该链配置时切换。  
- **EIP-3085 `wallet_addEthereumChain`**：**添加**新链（需链参数：`chainId/name/rpcUrls/nativeCurrency/blockExplorerUrls`），多数钱包会弹安全提示。

**前端策略**
1. 尝试 `switch`，若报错 `4902`（链不存在）→ 调 `add`。  
2. 仍失败 → 显示**手动切换指南**（链名/chainId/RPC/浏览器链接），并回退到**只读**模式。

```ts
async function ensureChain(provider, chain) {
  try {
    await provider.request({ method: 'wallet_switchEthereumChain', params: [{ chainId: chain.idHex }] });
  } catch (e:any) {
    if (e.code === 4902) {
      await provider.request({ method: 'wallet_addEthereumChain', params: [chain.meta] });
    } else throw e;
  }
}
```

---

## 5) SSR / 移动端适配：Provider 检测与 Deep Link
**SSR**
- 任何访问 `window/ethereum` 的代码必须在 `useEffect` 后执行或做 `typeof window !== 'undefined'` 守卫。
- 抽象 **“连接意图”** 与 **“实际 provider 就绪”**：SSR 渲染 skeleton，CSR 后根据持久化的“上次连接器”自动尝试恢复。

**移动端 Deep Link**
- WalletConnect：构造 `wc:` URI → 唤起钱包（Universal Link/自定义 scheme）。  
- 注入类钱包 App 内 WebView：`window.ethereum` 可能可用，但事件行为与桌面不同，需**统一去抖**处理。

**健壮检测**
```ts
const isBrowser = typeof window !== 'undefined';
const hasInjected = isBrowser && !!(window as any).ethereum && !(window as any).ethereum.isBraveWalletDisabled;
```

---

## 6) 签名协议矩阵：`eth_sign`/`personal_sign` vs EIP-712
**原理**
- `eth_sign`/`personal_sign`：对字节/文本作裸签（EIP-191 将文本前缀 `"\x19Ethereum Signed Message:\n"`），**不可表达域与结构**。  
- **EIP-712**：`domain (name, version, chainId, verifyingContract, salt)` + `types` + `message`，钱包能**显示结构化内容**并绑定域/链，抗重放。

**前端决策**
- 登录/授权/订单等**都用 EIP-712**；  
- 仅做“签字确认”或低风险点到点校验可用 `personal_sign`；  
- 禁止 `eth_sign`（很多钱包默认隐藏）。

**EIP-712 示例**
```ts
const domain = { name:'MyDApp', version:'1', chainId, verifyingContract: CONTRACT };
const types = { Permit:[{name:'owner',type:'address'}, {name:'spender',type:'address'}, {name:'value',type:'uint256'}, {name:'nonce',type:'uint256'}, {name:'deadline',type:'uint256'}] };
const message = { owner, spender, value, nonce, deadline };
const sig = await provider.request({ method:'eth_signTypedData_v4', params:[owner, JSON.stringify({domain, types, message, primaryType:'Permit'})] });
```

**UX 规范**
- 在签名前给予**可读摘要**（合约名/链/到期/额度）。  
- 对高风险方法（`approve/setApprovalForAll`）放**红色警示**与“可撤销”说明。

---

## 7) SIWE（EIP-4361）登录：挑战-响应与多钱包多链
**流程**
1. 前端向后端请求 **nonce**（绑定域/链/到期）。  
2. 生成 SIWE 文本（包含 address、chainId、statement、nonce、expirationTime）。  
3. 用户用 **EIP-712** 或 `personal_sign` 签名（推荐 712）。  
4. 后端 **恢复地址/或走 1271 校验** → 建立会话（cookie/token），返回登录成功。

**多钱包/多链**
- 会话**绑定 (chainId, address)**；切换账户/链时，前端**使会话失效**或 silent re-auth。  
- 断网/切换设备：凭 cookie/token 自动恢复，但遇到 `accountsChanged` 要**强提示重登**。

**伪代码**
```ts
const {nonce} = await fetch('/siwe/nonce').then(r=>r.json());
const msg = createSiweMessage({address, chainId, nonce});
const sig = await signer.signMessage(msg); // 或 712
await fetch('/siwe/verify', { method:'POST', body: JSON.stringify({msg, sig}) });
```

---

## 8) 合约账户签名（EIP-1271）：EOA/CA 混用场景
**原理**：合约账户不能“椭圆曲线恢复”，需调用合约 `isValidSignature(hash, signature)`，返回 **magic value** `0x1626ba7e` 代表有效。

**前端策略**
- 不要假设 `recoverAddress(sig) == signer`；先探测地址是否合约（`eth_getCode`）。  
- 若是合约 → 调其 1271；若不是 → 用 `ecrecover` 路线（库已封装）。

**验证示意（viem）**
```ts
import { isAddress, recoverMessageAddress, readContract } from 'viem';

async function verifySig(addr:string, hash: `0x${string}`, sig:`0x${string}`) {
  const code = await publicClient.getBytecode({ address: addr as any });
  if (!code || code === '0x') {
    return (await recoverMessageAddress({ message: hash, signature: sig })) === addr;
  }
  const ok = await readContract({ address: addr as any, abi:[{name:'isValidSignature',inputs:[{type:'bytes32'},{type:'bytes'}],outputs:[{type:'bytes4'}]}], functionName:'isValidSignature', args:[hash, sig] });
  return ok === '0x1626ba7e';
}
```

---

## 9) 链上支付（1）：EIP-681 支付 URI / 二维码
**格式**
```
ethereum:{address}[?value={wei}&gas={gas}&gasPrice={wei}&chainId={id}&function={sig}&parameters…]
例：ethereum:0xAbC...?value=10000000000000000   // 0.01 ETH
例：ethereum:pay-0xAbC@eip155:1?value=...       // CAIP 带链指明
```
**前端实践**
- **生成**：限制小数 → 转 wei；显式写 `chainId`；可附 `memo`（自定义键，由前端解释）。  
- **解析**：严格校验 `address/chainId/value` 与**目标代币**（ETH vs ERC-20）；对**超大金额/未知链**强警示。  
- **二维码**：用 `qrcode` 生成，移动端钱包可扫描唤起。

**安全**
- 永远在 UI 上同时显示 **明文地址** 与 **ENS 反解**（若有），防止二维码被替换。  
- 把“收款地址与链”作为**只读字段**传到支付确认页，禁止中途被脚本改写。

---

## 10) 链上支付（2）：ERC-20 + Permit（EIP-2612/Permit2）
**链路对比**
- 传统：`approve(spender, amount)`（交易1） → `contract.transferFrom(user, merchant, amount)`（交易2）。  
- **Permit**：用户**离线签名**授权（不上链） → 由 `spender` 在一次交易中消费该授权并完成 `transferFrom`（或由合约内部调用），**省掉交易1**。
  - **EIP-2612**：每个代币自带 `permit(owner, spender, value, deadline, v,r,s)`。  
  - **Permit2**（Uniswap）：统一授权合约，支持多 token 批量与细粒度限制。

**前端步骤（EIP-2612）**
1. 读取 token 的 **`nonces(owner)`** 与 `name`, `version`, `chainId`.  
2. 组装 712 域与 `Permit` 类型，用户签名。  
3. 由商户/合约在 `transferFrom` 前调用 `permit` 消耗授权（或路由合约内先 `permit` 再 `transferFrom`）。

```ts
// 1) 组装 712 并签名
const domain = { name, version:'1', chainId, verifyingContract: token };
const types = { Permit:[{name:'owner',type:'address'},{name:'spender',type:'address'},{name:'value',type:'uint256'},{name:'nonce',type:'uint256'},{name:'deadline',type:'uint256'}] };
const message = { owner, spender, value, nonce, deadline };
const sig = await signer.signTypedData({ domain, types, primaryType:'Permit', message });

// 2) on-chain：router.permitAndPay(sig, message) → 代扣
```

**安全与 UX**
- **默认额度最小化**（仅本次金额 + buffer），设置 **`deadline` 短时效**；  
- 显示**撤销入口**（`revoke.cash` 或内置“授权管理”页：读取 `allowance` 与历史 `permit`）；  
- 对“**无限授权**”与“**未知 spender**”给红色提醒；  
- 失败回退：不支持 2612 的 token → 回退为标准 `approve + transferFrom`，在 UI 上提示需多一次确认。

---
## 11) 交易费用（EIP-1559）：原理、估算、小费策略与替换交易
**费用模型**
- 区块目标气量 `targetGas`，当块气量 `gasUsed > targetGas`，**`baseFee` 上调**；反之下调（单块最大约 ±12.5%）。`baseFee` 被**销毁**，不能直接拿到矿工/验证者。
- 用户提交 **`maxFeePerGas`**（愿付上限）与 **`maxPriorityFeePerGas`**（小费/给出块者的激励）。实际支付：  
  `effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)`。

**前端可解释提示**
- 展示三段：`baseFee`（链上动态）+ `tip`（用户可调）= `effectiveGasPrice`（不会超过 `maxFee`）。  
- 风险提示：**尖峰期** `baseFee` 会跳涨；`maxFee` 太低会 pending；**替换交易**需要更高的 `tip` 与 `maxFee`。

**估算策略（前端）**
- 用 `eth_feeHistory` 拿最近 N 个区块的小费分位（如 p25/p50/p90），平滑为推荐 `tip`；`maxFee = tip + 2×nextBaseFee`（保守冗余）。
```ts
// viem 示例
const hist = await publicClient.request({
  method: 'eth_feeHistory',
  params: ['0x10', 'latest', [25, 50, 90]],
});
const nextBase = BigInt(hist.baseFeePerGas.at(-1)!);
const tipP50   = BigInt(hist.reward.map(r=>r[1]).map(BigInt).sort()[0] || 1n); // 近似
const maxPriorityFeePerGas = tipP50 > 1n ? tipP50 : 1_000_000_000n; // 至少 1 gwei
const maxFeePerGas = nextBase * 2n + maxPriorityFeePerGas;
```

**替换交易（加速/取消）**
- **同一 nonce** 重新发送；多数客户端要求 `newPriority ≥ oldPriority × (1 + bump)`（常见 10%–12.5%），同时 `newMaxFee ≥ newBase + newPriority`。  
- **取消**：发送一笔 **to=自己、value=0** 的同 nonce 交易，费用更高，抢先被打包即“占位”取消。
```ts
await walletClient.sendTransaction({
  account, to: account, value: 0n, nonce: oldNonce,
  maxPriorityFeePerGas: oldTip * 12n/10n,
  maxFeePerGas: (await publicClient.getGasPrice()) + oldTip * 12n/10n
});
```

---

## 12) 交易生命周期：模拟 → 估气 → 发送 → 等待 → 确认（含异常分支）
**状态机**
```
Idle → (prepare) → Simulating → Estimated → Sending → Pending(on-chain)
  ├─ Confirmed(k>=target)         // 达到确认数
  ├─ Dropped/Replace              // 被替换/淘汰
  └─ Failed(revert/OOG/underpriced)
```

**关键步骤**
1) **模拟 (`eth_call`)**：在**当前状态**下执行，提前暴露 revert 原因（自定义错误/require）。  
2) **估气 (`eth_estimateGas`)**：给出 upper bound；失败常见因为权限/余额/路径错误。  
3) **发送**：带 1559 参数；拿到 `txHash` 进入 `pending`。  
4) **等待 (`getTransactionReceipt`)**：轮询或 ws 订阅；拿到 `status`/logs/gasUsed。  
5) **确认数**：等待 k 个确认（或 `finalized`/`safe` 标签）以抗短 reorg。

**异常处理**
- **`UNPREDICTABLE_GAS_LIMIT`**：模拟失败，拦截并显示**可读错误**（见第 16 题）。  
- **`replacement underpriced`**：替换费率不足；提示“一键加速”并提高 tip。  
- **超时**：给出“继续等待 / 提高小费 / 取消”选项。

```ts
// viem 最小闭环
const data = encodeFunctionData({ abi, functionName, args });
await publicClient.call({ to: address, data }); // 模拟
const gas = await publicClient.estimateGas({ account, to: address, data });
const hash = await walletClient.sendTransaction({ account, to: address, data, gas, maxFeePerGas, maxPriorityFeePerGas });
const rec  = await publicClient.waitForTransactionReceipt({ hash, confirmations: 2 }); // 2~5 视场景
```

---

## 13) 只读 / 可写分层：Provider vs Signer 与多链防错
**原则**
- **读（public client）**：走稳定 RPC，容错/重试友好；**不依赖用户钱包**。  
- **写（wallet client）**：仅在用户交互后创建；与当前连接的钱包/链绑定。

**多链防错**
- 所有合约地址表按 **`chainId → name → address`** 管理；调用前**双重校验**（当前链与地址表链一致）。  
- 合约 ABI 一处维护，**版本化**（升级代理 ABI 不兼容时按版本选择）。

```ts
const addresses = {
  1:   { router: '0x...', token: '0x...' },
  137: { router: '0x...', token: '0x...' },
} as const;

function addr(chainId:number, key: keyof typeof addresses[1]) {
  const a = (addresses as any)[chainId]?.[key];
  if (!a) throw new Error(`No ${key} on chain ${chainId}`);
  return a;
}
```

---

## 14) 批量读取（Multicall）：原理、分页与部分失败
**协议要点**
- Multicall 合约把多条 `staticcall` 打包一次执行：  
  - **`aggregate`**：任一失败会**整体 revert**。  
  - **`tryAggregate`**：每项返回 `{success, returnData}`，允许**部分失败**。

**前端策略**
- **分页**：单次调用数量控制在几百条内（返回 data 过大会超时/限流）。  
- **解码**：对 `success=false` 的项给默认值/错误标记；对 `returnData` 用 ABI 解码。  
- **去重**：相同 `(address,selector,args)` 合并。

```ts
// viem: 使用 multicall 内置（链需内置 multicall 地址）
const res = await publicClient.multicall({
  contracts: [
    { address: token, abi: erc20, functionName: 'balanceOf', args: [acct] },
    { address: token, abi: erc20, functionName: 'allowance', args: [acct, spender] },
  ],
  allowFailure: true,
});
```

---

## 15) 日志与事件：`eth_getLogs` 的窗口、过滤与重组处理
**过滤原理**
- 节点通过 **布隆过滤**预筛事件，再精确匹配 `address/topics`。大窗口 + 多地址会被限流/超时。

**窗口策略**
- **分段**：按块范围分页（如每次 2k–10k 块，依链负载调整），失败则减半回退。  
- **去重键**：`txHash + logIndex`；多页合并保持幂等。  
- **最终性**：优先用 `finalized`/`safe` 标签；否则保留“**回滚区**”（最近 N 块）重复拉取并对比差异。

```ts
const PAGE = 5_000n;
for (let from = start; from <= end; from += PAGE) {
  const to = from + PAGE - 1n;
  const logs = await publicClient.getLogs({ address, topics, fromBlock: from, toBlock: to });
  mergeDedup(logs); // key = txHash+logIndex
}
```

---

## 16) 索引取舍：链上直读 vs The Graph/Substreams；本地缓存与回放校准
**选型**
- **直读 (`eth_getLogs`)**：**一致性强**、延迟受节点影响；复杂聚合需前端/后端自行计算。  
- **The Graph/Substreams**：**延迟低、聚合好**；但受索引节点新鲜度与**重组一致性**策略影响。

**前端一致性方案**
- 维护 **“区块水位线”**：页面有两层数据源  
  1) **索引层**：读 Graph，块高为 `H_idx`；  
  2) **校准层**：用 `getLogs` 从 `H_idx - window` 到 `latest` 回放增量，与本地缓存**比对/幂等合并**。  
- 存储：IndexedDB/localStorage 保留 `lastSyncedBlock` 与日志快照，避免刷新全量重扫。

**失败与回退**
- Graph 502/延迟：降级为全链 `getLogs`；展示“索引延迟”的黄条提示。  
- 解析错误：把事件 schema 与解码异常上报，防止静默数据错位。

---

## 17) 代币交互：`decimals/allowance/approve/transferFrom` 的边界与竞态
**常见边界**
- `decimals` 必须在首次渲染读取并**缓存**；金额与 `BigInt`/十进制换算保持**精度**。  
- `allowance(owner, spender)` 是**每个 spender 独立**，更换合约要重查。  
- `approve` 竞态：用户发了 `approve(A, amt1)` 未上链，又发了 `approve(A, amt2)`，**后者会覆盖前者**；UI 应**串行化**与 loading 锁。

**最小安全流程**
1. 读取 `allowance`；若不足 → 引导 `approve`（最小额度，或一次性授权由用户选择）。  
2. 发送业务交易 `transferFrom`/路由器方法。  
3. 失败回滚：若第二步失败且第一步放了“大额授权”，提示可撤销并提供入口。

```ts
// safe approve：仅当 allowance < need 时设置到 need（或 need*1.1）
if (allowance < need) {
  const hash = await walletClient.writeContract({
    address: token, abi: erc20, functionName:'approve', args:[spender, need]
  });
  await publicClient.waitForTransactionReceipt({ hash });
}
```

---

## 18) NFT 元数据：`tokenURI`、IPFS/Arweave 网关、占位/缓存/敏感内容
**解析流程**
- `tokenURI(tokenId)` 可能返回 `ipfs://{cid}`、`ar://...`、`data:application/json;base64,...` 或 `https://...`。  
- **网关适配**：`ipfs://` → `https://{gateway}/ipfs/{cid}`；**多网关容错**（主→备）。  
- 元数据字段：`name/description/image/animation_url/attributes`；`image` 也可能是 `ipfs://` 或 `data:`。

**前端策略**
- **占位与渐进**：先渲染骨架/模糊图；加载失败切换备选网关。  
- **缓存**：`ETag`/`Cache-Control`，或 IndexedDB 存储已解析的 JSON 与图片 blob（控制上限/逐出）。  
- **敏感内容**：合约/集合黑名单、本地开关“显示敏感内容”，默认遮罩。  
- **安全**：禁止把富文本 HTML 直接注入；必要时用安全白名单渲染 Markdown。

```ts
function ipfsToHttp(uri: string) {
  if (uri.startsWith('ipfs://')) return `https://cloudflare-ipfs.com/ipfs/${uri.slice(7)}`;
  return uri;
}
const meta = await fetch(ipfsToHttp(await readTokenURI(id))).then(r=>r.json());
const img  = ipfsToHttp(meta.image || meta.animation_url);
```

---

## 19) ENS 与 CCIP-Read：解析/反解、链下读取与降级
**ENS 基础**
- **namehash**：递归 keccak 域名得到 32 字节根哈希，用于 Registry/Resolver 查询。  
- **解析**：`resolver(addr)` → 调 `addr(bytes32)`/`text(bytes32,key)` 等。  
- **反解**：地址对应的 `reverse` 记录 → 校验 `addr(namehash(name)) == address` 才可信。

**CCIP-Read（EIP-3668）**
- 当 Resolver 返回特殊错误 `OffchainLookup`（带回调 URL 与回调数据），钱包/客户端应去 **链下 fetch**，拿到证明后再回合约完成验证。  
- **前端提示**：这是“链下数据，经合约验证”；若失败/超时，提供**链上兜底**或“无法解析”的明确提示。

```ts
// viem 已内置 ccip-read 处理；如禁用：ccipRead: false
const resolver = await publicClient.getEnsResolver({ name: 'vitalik.eth' });
const addr     = await publicClient.getEnsAddress({ name: 'vitalik.eth' }); // 可能触发 CCIP-Read
```

**降级策略**
- 多解析尝试：主要 RPC 超时 → 备用 RPC；CCIP-Read 失败 → 显示“未验证”并允许用户重试。  
- 明确标注**来源**：on-chain / off-chain（CCIP），提升信任透明度。

---

## 20) L2 费用与 EIP-4844：组成、估算与展示
**费用拆分**
- **L2 执行费**：在 L2 上执行指令的 gas × `gasPrice(L2)`。  
- **L1 数据费**：把交易/批次数据发布到 L1 的成本（与数据字节数相关）。  
- **EIP-4844（Blob）**：为大数据提供**独立的 Blob Gas 市场**，显著降低 DA 成本；Blob 数据**不进入以太坊历史状态**，只保证可用性窗口。

**对前端的影响**
- 估算/展示要把 **L2 执行** + **L1 数据** 分开说明（不同 L2 的 RPC 字段不同：有的在回执里给 `l1Fee`，有的提供额外方法/预估接口）。  
- 文案解释：  
  - “在 L2 执行消耗 X”  
  - “为在 L1 发布数据消耗 Y（受 4844 Blob 价格影响）”。  
- 结算后对账：读取回执中的 **实际 `gasUsed`**、**effectiveGasPrice**，以及（若可得）**`l1Fee`/`blobFee`**，给出“预估 vs 实际”的差异。

**估算骨架（伪代码）**
```ts
// 1) 估 L2 执行
const gas = await publicClient.estimateGas({ ...tx });
const gasPrice = await publicClient.getGasPrice();
const execCost = gas * gasPrice;

// 2) 估 L1 数据（依 L2 提供的端点，示意）
const l1 = await publicClient.request({ method: 'rollup_estimateL1Fee', params: [tx] }).catch(()=>0n);

// 3) 汇总展示
show({ execCost, l1DataCost: l1, total: execCost + l1 });
```

**风险提示**
- 热点时段 L1/Blob 费波动较大，**预估不等于最终**；允许用户选择“慢/标准/快”并解释差异。  
- 有的 L2 会在结算后**退还**未用的 L1 费，前端需在“交易详情”展示**最终费用**与退款。

---
## 21) 私有交易与 MEV 防护：原理、权衡与前端落地
**公开 mempool vs 私有内存池**
- 公开 mempool：交易在被打包前可见，可能被三明治/抢跑。
- 私有内存池（Flashbots Protect、MEV-Blocker 等）：交易**直接送给 Builder/Relay**，不经公共 mempool；若交易未入块，不会泄露内容。

**用户感知的权衡**
- 优点：降低被夹击概率；支持“中继保护”（不让矿工提前看）。
- 风险/代价：确认可能**更慢**；未入块前**无公共 txhash**，状态追踪依赖中继回执；部分钱包/节点不支持私有端点。

**前端设计**
- 提供“私有/公开”开关；默认公开，遇到 DEX/拍卖等**易被夹击操作**时建议开启。
- 明确文案：私有交易可能无法在常见浏览器立即查询；显示**中继跟踪链接**或“等待入块后生成 txhash”。

**实现（切换 RPC / 发送私有交易）**
```ts
import { createWalletClient, http, custom } from 'viem';

// 公开 RPC（默认）
const publicWallet = createWalletClient({ chain, transport: custom(window.ethereum) });

// 私有 RPC（Flashbots Protect/MEV-Blocker，有的支持标准 sendTransaction）
const privateWallet = createWalletClient({ chain, transport: http('https://rpc.flashbots.net') });

// 发送：仅在用户选择私有时走 privateWallet
const hash = await (usePrivate ? privateWallet : publicWallet).sendTransaction({
  account, to, data, value, maxFeePerGas, maxPriorityFeePerGas, // 1559 参数仍需
});
```
> 有些私有服务提供 `eth_sendPrivateTransaction` 专有方法或 header，需要适配器；失败应**回退公开发送**（用户可确认）。

---

## 22) 跨链桥 UX：消息桥/流动性桥的阶段化进度与回退
**两类桥**
- **消息桥（官方/证明型）**：L2→L1 常见流程：`提交（L2）→ 证明/挑战窗口 → 最终执行（L1）`；时间从分钟到数天不等（Optimistic 需等待期）。
- **流动性桥（LP / AMM）**：对手方先行垫资，跨链**近实时**，但有费率/滑点与可用流动性限制。

**前端进度模型**
```
Submitted (src) → Proving (src) → RelayScheduled (dst) → Executed (dst)
失败分支：ProofExpired / RelayFailed / Timeout → 提示重试或人工申领
```

**实现骨架（同时订阅两条链）**
```ts
// 1) 源链发送并拿到 srcTxHash
track(srcChain, srcTxHash, ['Sent','Proved']);

// 2) 从桥 API 或合约事件拿到目标链待执行任务（或中继的中间交易哈希）
track(dstChain, relayTxHash, ['Relayed','Executed']);

// 3) 统一在 UI 显示阶段与预计剩余时间（基于历史均值）
```

**UX 要点**
- 清晰显示两条链的**状态条**与“预计时间/费用拆分”（L1/L2/桥费）。
- 失败可**重试/更换中继**；提供**人工申领**（某些桥需用户触发执行）。
- 给出**透明链接**（两链浏览器 + 桥状态页），避免用户焦虑。

---

## 23) 多签（Safe）适配：阈值签名、批处理与队列可视化
**原理**
- Safe 合约维护**阈值**（m-of-n）。一笔交易需达阈值后由任一成员**执行**。
- **Transaction Service** 索引挂起/已执行交易；**Safe Apps SDK** 在 dApp 中发起/签名。

**前端操作流**
1. 连接 Safe（检测 `isSafeApp` 或注入的 provider 标识）。
2. 构造待执行交易（单笔或 **batched** 多操作）。
3. 发起签名 → 上传到 Tx Service（不立即上链）。
4. 显示**签名进度**（哪些 owner 已签）与**模拟结果**。
5. 达阈值后显示“执行”按钮；执行回执写回 UI。

**代码片段（概念示意）**
```ts
import SafeAppsSDK from '@safe-global/safe-apps-sdk';
const appsSdk = new SafeAppsSDK();

// 1) 获取 Safe context
const { safeAddress, chainId } = await appsSdk.safe.getInfo();

// 2) 准备交易（批处理）
const txs = [{ to: token, value: '0', data: approveData }, { to: router, value: '0', data: swapData }];
const { safeTxHash } = await appsSdk.txs.send({ txs });

// 3) 轮询签名进度或订阅 service 事件，展示队列
```
**UX 重点**
- 将“已签/未签/执行人”清晰罗列；支持**复制请求链接**发给其他签名者。
- 在“模拟通过/失败”上加醒目标识，避免把必失败交易推进到链上。

---

## 24) 账户抽象（EIP-4337）：UserOperation/Paymaster/Session Key 的前端流
**协议关键角色**
- **UserOperation (UO)**：类似交易的打包对象（包含调用、Gas、签名字段）。
- **Bundler**：收集 UO 并经 **EntryPoint** 合约执行。
- **Paymaster**：代付 Gas 或限制策略（白名单/配额/签名）。

**前端交互流**
```
prepare UO → sign UO → sendUserOperation (bundler RPC) → getUserOperationReceipt → handled (on-chain)
```
- **回退策略**：钱包/链不支持 AA 时，回退为 EOA `sendTransaction`。

**示意（基于通用 AA SDK 思路）**
```ts
// 通过 SDK 构造 smartAccount
const sa = await createSmartAccount({ owner: eoaSigner, chain });
// 发送 UO（内部会签名并调 bundler）
const uoHash = await sa.sendUserOperation({ to, data, value });
const receipt = await sa.waitForUserOperationReceipt(uoHash);
```

**UI/安全**
- 明确提示“**由某某 Paymaster 代付** / **Session Key 限权**（仅限合约 X 的方法 Y，额度/到期 Z）”。
- 显示**UO → 最终交易**的对应关系（部分浏览器已支持 UO 索引；否则提供 bundler explorer 链接）。

---

## 25) 自定义错误与 Revert 解码：从 `data` 到可读提示
**原理**
- Solidity Custom Error 编码：`Error(selector = keccak("ErrorName(type,...)")[0..4]) + abi.encode(args)`。
- 节点在 revert 时将编码放入回执/异常 `data` 字段。

**前端解码（viem）**
```ts
import { decodeErrorResult } from 'viem';

try {
  await publicClient.call({ to, data }); // 模拟
} catch (e: any) {
  const { errorName, args } = decodeErrorResult({ abi, data: e.data });
  // 映射到用户文案
  show(mapError(errorName, args));
}
```
**文案映射**
- 建立**错误字典**（`InsufficientAllowance` → “额度不足，请先授权 X”）。
- 对**参数化错误**（如 `MinOut(uint256 expected, uint256 got)`）在提示中展示关键数值。

---

## 26) 价格与预言机：Chainlink/TWAP 的前端读取与防操纵提示
**Chainlink Aggregator**
- 读取：`latestRoundData()` → `{ answer, updatedAt }`；注意 **decimals**。
- 防陈旧：若 `now - updatedAt > threshold`（例如 30 分钟）则标红并**阻断关键操作**。

```ts
const [ans, dec, rd] = await Promise.all([
  readContract({ address: agg, abi, functionName: 'latestAnswer' }),
  readContract({ address: agg, abi, functionName: 'decimals' }),
  readContract({ address: agg, abi, functionName: 'latestTimestamp' }),
]);
const price = Number(ans) / 10 ** Number(dec);
const stale = Date.now()/1000 - Number(rd) > 30*60;
```

**TWAP/Median**
- 前端可读取去中心化交易所的 **时间加权平均价**（TWAP）或使用中位数喂价合约。
- 在下单页提示：价格采用 **TWAP/预言机**，可降低闪电贷操纵风险；同时展示**即时价**与**保护价**差异。

---

## 27) 隐私与零知识：证明生成/提交的前端策略
**核心难点**
- 证明生成（SNARK/STARK）在浏览器**计算量大/内存大**；电路参数（.zkey/.wasm）体积大。
- 选择：**本地生成**（隐私最佳，但耗时）或**远端生成**（快，但需信任/抗滥用）。

**前端工程化**
- 资源分发：参数文件 **分块/按需加载**（CDN + integrity 校验）；首次使用预热缓存。
- **Web Worker** & 进度回调：避免主线程卡死，实时显示百分比/预计时间。
- 失败可诊断：区分**证明无效**（输入不一致）、**资源加载失败**、**内存不足**三类错误。

```ts
// 伪代码：worker 里跑 prove()
const worker = new Worker(new URL('./zk-worker.ts', import.meta.url));
worker.postMessage({ circuit: 'transfer', inputs });
worker.onmessage = ({ data }) => { if (data.type==='progress') setPct(data.pct); else submitProof(data.proof); };
```

**提交与验证**
- 发送交易前**本地 verify**（若提供 verifier wasm）；失败提前拦截。
- UI 显示：证明大小、链上验证费、隐私声明与信任假设（有无可信设置）。

---

## 28) 多链地址簿与 ENS：校验、头像与链绑定
**地址校验**
- EIP-55 校验和（大小写编码）；录入时对不匹配的地址**告警**。
- 与链绑定：保存为 `(chainId, address)` 对，防止把主网地址误用于 L2。

**ENS**
- 正向解析：`resolve(name) -> address`；反向解析：`reverse(address) -> name` 并校验**前后匹配**才显示徽标。
- 头像：读取 `text(name, 'avatar')`；IPFS/HTTPS 网关转换与缓存。

```ts
function validateChecksum(addr: string) {
  return isAddress(addr, { strict: true }); // viem 严格校验和
}
```

**UX**
- 输入框支持 `ENS/地址` 混输；解析成功后锁定为**地址**并显示昵称/头像。
- 地址簿持久化：本地 + 云（可选），对隐私敏感数据加密存储。

---

## 29) 速率限制与容错：多 RPC、超时与幂等重试
**问题场景**
- 公共 RPC 在高峰期**限流/超时**；节点实现差异导致**偶发错误**（如 `header not found`）。

**工程策略**
- **Fallback Provider**：多 RPC 加权/优先级 + **超时切换**。
- **指数退避**：对可重试错误（5xx/timeout）采用 `base * 2^n` 退避；对链上确定性错误（revert）**不重试**。
- **幂等**：只读请求用**去重键**（相同参数短时命中内存缓存）；写请求靠 **nonce** 天然幂等。

```ts
import { fallback, http } from 'viem';
const transport = fallback([
  http('https://rpc.primary', { batch: true, timeout: 8_000 }),
  http('https://rpc.backup',  { batch: true, timeout: 8_000 }),
], { rank: true, retryCount: 2 });
```

**UI**
- 显示“已自动切换备用节点”；提供**重试按钮**与错误快照（请求参数/区块高度）以便用户或客服诊断。

---

## 30) 安全签名 UX：高风险方法的前置解释与防误签
**高风险清单**
- `approve/setApprovalForAll/increaseAllowance`（授权滥用）；  
- `permit/permit2`（离线授权被盗用风险）；  
- `delegateBySig` 等治理委托；  
- 任意 `call`/`multicall`（聚合器可能夹带恶意方法）。

**前置提示（签名弹窗前）**
- 明示：**链/合约名/方法/额度/到期/spender**；对“无限授权”用红色标签，默认建议**限额+短有效期**。
- 链接到 **撤销授权** 页面；签名后展示“如何撤销”的帮助。

**防误签实现**
- 在本地对目标合约 ABI 做**白名单匹配**，未知合约或未识别方法 → 显示“高风险”。
- 对 EIP-712：把 `domain`/`verifyingContract`/`chainId` 渲染在 UI；对 `salt/nonce/deadline` 做合理性校验。
- 对 `personal_sign`：限制为**特定格式**（如 SIWE）并在 UI 呈现文本摘要。

```ts
function renderApprovalSummary({ token, spender, amount, deadline }) {
  return [
    `代币：${token.symbol} (${token.address})`,
    `授权给：${spender}`,
    `额度：${formatAmount(amount)}（建议限额）`,
    `到期：${formatDeadline(deadline)}`,
  ].join('\n');
}
```

**提交后回看**
- 交易详情页展示**最终授权状态**（allowance 实时查询）；若用户取消/替换失败，提示**进一步操作**（提高小费/撤销授权）。

---
## 31) 交易模拟（Tenderly / Anvil / Foundry Fork）：前端集成与可阻断失败分类

**原理分层**
- **本链模拟**：`eth_call` 在给定 `blockTag`（`latest/pending/finalized`）的**当前状态**下执行交易，不写状态、但能返回 revert 数据（字符串/自定义错误编码）。  
- **Fork 模拟**（开发/灰度）：把主网状态**分叉**到本地（Anvil/Hardhat/Foundry），支持“时间前进/余额注入/账户模拟”。  
- **第三方沙盒**：Tenderly/Blockscout 模拟 API，可在**远端**按指定区块与余额、`stateOverrides` 运行并产出更详细的 trace。

**前端最佳实践（上线环境）**
1) **离链前置检查**：余额/授权/截止时间/路径可达（如 DEX 报价）等“静态条件”。  
2) **`eth_call` 干跑**（dry-run）：对最终 `to/data/value` 直接调用，拦截会必然失败的错误。  
3) **可读错误**：对 `e.data` 解码**自定义错误**与 `Error(string)`，映射成 UI 文案。  
4) **边界选择**：对可能受“**竞争态**”影响的逻辑（例如池子价格变化），把干跑作为**建议**而非强约束；对**权限/额度/超时**类失败，**直接阻断**。

**可阻断失败（应阻断）**
- `InsufficientAllowance/Balance`、`DeadlineExpired`、`Paused/OnlyOwner`、`SignatureExpired/Invalid`、`SlippageTooHigh（保护价检查）`、`UnsupportedToken/Chain`。

**可提示失败（不阻断，仅提示风险）**
- **价格/深度波动**导致的实际执行差异（干跑与真实打包之间有竞态）。  
- **外部依赖**（预言机延迟、桥跨域执行）引起的后续失败。

**示例（viem，干跑 + 解码错误）**
```ts
import { encodeFunctionData, decodeErrorResult } from 'viem';

async function dryRun({ to, abi, functionName, args, value }) {
  const data = encodeFunctionData({ abi, functionName, args });
  try {
    await publicClient.call({ to, data, value }); // eth_call
    return { ok: true };
  } catch (e: any) {
    // Ethers/viem 都能取到 e.data（0x...）
    try {
      const { errorName, args: params } = decodeErrorResult({ abi, data: e.data });
      return { ok: false, code: errorName, params };
    } catch {
      return { ok: false, code: 'UnknownRevert', raw: e.data };
    }
  }
}
```

**Tenderly（可选）**
- 使用 Tenderly 模拟 API 发送 `from/to/value/input` 与 `state_objects` 覆盖余额/nonce，拿到 trace 与 gas 估计；前端只在**内部环境**或**灰度回放**中使用，线上需注意**隐私**与**速率**。

---

## 32) 链上支付风控（电商收款/订阅扣费）：交互设计与防骗要点

**单次收款（一次性支付）**
- **EIP-681 URI**：`ethereum:0xRECIPIENT@eip155:1?value=...&token=0xTOKEN&amount=...&memo=...`  
  - 前端生成/解析时**强校验**：`chainId/收款地址/token`；金额转 `wei`；附 `memo`（订单号）。  
  - UI 同屏展示**地址（前 6 / 后 4）+ ENS（若有）+ 链名**；二维码下方**明文**展示这三者，防替换。
- **最小收到**（保护价）：对 ERC-20/DEX 交易，计算 `minAmountOut = quote * (1 - slippage)`，签名前明确写入参数与**截止时间**。
- **回执对账**：支付完成后查询 `receipt.logs` 核对（`to/tokenId/amount`），失败时给出**标准救济流程**（联系客服/退款申请）。

**订阅/代扣（循环支付）**
- **授权模式**：  
  - `approve(spender, limit)` + 周期性 `transferFrom`。  
  - **EIP-2612/Permit2**：每一期**短期限额 + 到期**的离线授权，合约在本期消费后失效（推荐）。  
- **额度策略**：默认给**本期金额 + 余量**（如 5%），而非无限授权；明确展示“已授权额度 / 剩余额度 / 到期时间”。  
- **取消路径**：订阅详情页提供**一键撤销授权**（读取 `allowance` → 发 `approve(spender, 0)` 或 Permit2 `revoke`），并提示生效时间。

**反钓鱼与混淆代币**
- **链/地址锁定**：支付页的链与收款地址在**路由参数**与**签名文本**双重绑定；路由变化需用户复核（红框警示）。  
- **代币校验**：从**白名单 TokenList**（自有或可信源）匹配 `symbol/decimals`；`getCode(addr)=='0x'` 或与白名单不一致 → 红色警示“未知代币”。  
- **域绑定**：商户对“收款参数 JSON（address/chain/token/amount/nonce/exp）”做 **EIP-712** 签名；前端验证签名人（商户公开地址），失败不允许支付。

---

## 33) 代币列表安全：Token List 信任链、哈希校验与“未知代币”强提示

**Token List（Uniswap 标准）要点**
- 结构：`name/version/timestamp/tokens[]`（`{ chainId, address, name, symbol, decimals, logoURI }`）。  
- 版本：`major/minor/patch`；前端可做**版本差异审计**（新增/变更/删除）。  
- 来源：HTTPS/CORS，可配合**子资源完整性（SRI）**或**发布者签名**（部分团队扩展用 712/PGP）。

**前端安全策略**
- **来源白名单**：仅允许来自受信发行者的列表（域名/公钥）；同时支持**本地镜像**（防源站变更）。  
- **哈希锁**：首次拉取后记录 **content hash**；仅当哈希匹配或签名验证通过才自动更新，否则要求用户手动确认。  
- **合约存在性**：拉取到的地址，首次使用前做 `getCode(addr)` 检查，不存在→拒绝。  
- **冲突检测**：同一 `symbol/decimals` 出现多个地址 → 在 UI 显示“潜在仿冒”，默认不展示。  
- **未知代币强提示**：当用户粘贴一个未在列表中的地址：  
  - 显示“未知代币”红条；  
  - 展示 `symbol/decimals` 读取结果与**合约创建者/持币分布**（若接入区块浏览器 API）；  
  - 要求用户**勾选确认**后才允许继续。

**最小实现（校验与落盘）**
```ts
const list = await fetch(LIST_URL, { integrity: SRI_HASH }).then(r=>r.json());
assertValidSchema(list);
const hash = sha256(JSON.stringify(list.tokens));
localStorage.setItem('tokenlist.hash', hash);
```

---

## 34) SDK 选型与包体：ethers v5/v6、viem、wagmi/RainbowKit；懒加载 ABI/链配置与避免 Node polyfill

**对比**
- **ethers v5**：CJS 为主、内部子模块耦合，tree-shaking 一般；常引入 `bn.js`、`hash.js`，包体偏大。  
- **ethers v6**：ESM、原生 `bigint`，更易摇树；API 变化较大。  
- **viem**：ESM、函数式、按需导入、`bigint`；与 **wagmi** 深度集成；浏览器友好。  
- **wagmi**：连接器 + 状态管理，配合 **RainbowKit** 提供 UI；SSR 支持好。  
- **RainbowKit**：钱包选择与主题组件，易上手；注意按需导入减少主题资产体积。

**包体控制**
- **仅浏览器 API**：避免引入 Node polyfill（`crypto/stream/buffer`）。在 Next/Vite 中：  
  - 禁止误用 Node 模块；ethers v5 若触发 polyfill，考虑 v6/viem。  
  - `resolve.alias` 明确到浏览器实现或**不提供**（防 bundler 自动注入）。
- **按路由懒加载**：  
  - 大型 ABI/字典用 `import()` 动态加载（e.g., 交易页才加载 AMM 路由 ABI）。  
  - 多链配置拆分：当前链仅加载当前链 `chain` 对象与地址表。  
- **摇树与最小导入**：`import { createPublicClient } from 'viem'`（不要 `import * as viem`）；wagmi 只引入用到的连接器。
- **图像/字体**：RainbowKit 主题资产做 CDN/懒加载；钱包图标走按需。

**示例（Next/Vite 禁 polyfill + 动态 ABI）**
```ts
// vite.config.ts
resolve: {
  alias: { util: false, stream: false, crypto: false } // 若库错误引 Node 依赖
}

// 动态加载 ABI
const { abi: routerAbi } = await import('../abis/uniswapv3_router.json');
```

---

## 35) 端到端最小 DApp（交互与安全清单）：多钱包连接 → EIP-712 登录 → Multicall 余额 → Permit 代扣 → 1559 费用 / 私有 RPC → 事件追踪（抗重组）

**页面状态机（核心）**
```
Boot → DetectEnv(SSR守卫)
 → Wallet[Idle|Connecting|Connected{provider,account,chain}]
   → Auth[None|Signing|LoggedIn{addr,chain,session}]
     → Ready
       → Tx[Preparing|Simulating|Estimating|Pending{hash}|Confirmed|Failed|Replaced]
```

**模块分层**
- **wallet**：EIP-1193 连接器（Injected/WC），事件驱动；只读 `publicClient` 与可写 `walletClient` 分层。  
- **auth**：SIWE（EIP-4361）+ 服务器会话，绑定 `(address, chainId)`。  
- **contracts**：`chainId → { name → address }` 的地址表 + ABI 动态导入；版本化（代理升级）。  
- **tx**：统一的发送器（干跑→估气→发送→等待），内置 1559 策略、替换交易与私有 RPC 选项。  
- **indexing**：先用 The Graph 拉列表，再用 `getLogs` 做回放校准（区块水位线）。

**关键交互流**
1) **连接与切链**：EIP-3085/3326；不支持→只读模式。  
2) **登录**：拉 nonce → 712 签名（SIWE）→ 建会话；`accountsChanged/chainChanged` → 会话失效或静默重登。  
3) **批量读取余额**：Multicall `balanceOf/allowance`；分页与 `allowFailure:true`。  
4) **支付（Permit 代扣）**：  
   - 读取 `nonces/name/version` → 712 `Permit` 签名（限额+短到期）。  
   - 合约 `permit + transferFrom` 一气呵成（或 Permit2）。  
5) **费用与发送**：  
   - 1559：`eth_feeHistory` 估 tip，`maxFee = nextBase*2 + tip`；  
   - 可选**私有 RPC**（Flashbots/MEV-Blocker）开关；失败回退公开。  
6) **追踪与重组**：等待 `k` 确认或 `finalized/safe` 标签；列表去重键 `txHash+logIndex`；最近 N 块回放。

**安全清单（签名与展示）**
- 签名前展示：合约名/链/方法/额度/截止/spender（高风险标红）；SIWE 文本中**域/链/过期**清晰可见。  
- 代币白名单 + `getCode` 检查 + 未知代币强提示；ENS 解析与**反解校验**一致才显示昵称。  
- 交易失败分类：用户取消 / 链上失败（可读错误）/ 网络错误（自动退避 + 切备用 RPC）。  
- 撤销入口：授权管理页读取 `allowance` 与 Permit2 记录；提供“一键 revoke”。

**骨架代码（极简集成）**
```ts
// clients.ts
export const publicClient = createPublicClient({ chain: mainnet, transport: fallback([http(A), http(B)]) });
export const walletClient  = createWalletClient({ chain: mainnet, transport: custom(window.ethereum) });

// siwe.ts
export async function login(addr:string, chainId:number) {
  const { nonce } = await api('/siwe/nonce');
  const msg = siweMessage({ domain: location.host, address: addr, chainId, statement: 'Sign in', nonce });
  const sig = await walletClient.signMessage({ account: addr as any, message: msg });
  return api('/siwe/verify', { method:'POST', body:{ msg, sig }});
}

// tx.ts
export async function sendTx(input) {
  await publicClient.call(input); // dry-run
  const gas = await publicClient.estimateGas(input);
  const { maxFeePerGas, maxPriorityFeePerGas } = await suggest1559(publicClient);
  const hash = await walletClient.sendTransaction({ ...input, gas, maxFeePerGas, maxPriorityFeePerGas });
  return await publicClient.waitForTransactionReceipt({ hash, confirmations: 2 });
}
```

**可观测性**
- 统一埋点：连接来源/链切换/签名类型/失败码；对 712/Permit 失败单独计数。  
- 错误脱敏：不上传私钥、助记词、签名原文；保留 `txHash/chainId/errorCode` 即可。

---
