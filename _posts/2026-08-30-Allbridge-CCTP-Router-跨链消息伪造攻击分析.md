---
tags:
  - CrossChainMessageFault
title: "Allbridge CCTP Router 跨链消息伪造攻击分析"
---

2026 年 8 月 19 日，跨链桥 Allbridge 的 Base 侧 CCTP Router 被攻击，损失约 <mark>19 万美元</mark>。这次事故的特别之处在于：攻击者手里那条跨链消息，是 Circle 亲自签名认证过的**合法**消息——只不过它对应的 100 万 USDC 从来没有被销毁过。

<style>
.posts .entry li,
.posts .entry a {
  overflow-wrap: anywhere;
  word-break: break-word;
}

.cctp-timeline {
  margin: 1.5rem 0;
  padding: 0;
  list-style: none;
  border-left: 2px solid var(--border-strong, #d3dae7);
}

.cctp-timeline > li {
  position: relative;
  margin: 0 0 1.35rem;
  padding: 0 0 0 1.25rem;
  list-style: none;
}

.cctp-timeline > li::before {
  content: "";
  position: absolute;
  left: -7px;
  top: 0.45em;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: var(--accent, #5865e0);
  box-shadow: 0 0 0 3px var(--bg, #fff);
}

.cctp-timeline .cctp-when {
  display: block;
  font-size: 0.85em;
  letter-spacing: 0.03em;
  color: var(--muted, #78849a);
}

.cctp-timeline .cctp-what {
  display: block;
  font-weight: 600;
  color: var(--heading, #1b2333);
}

.cctp-ledger {
  width: 100%;
  margin: 1rem 0;
  border-collapse: collapse;
  font-size: 0.94em;
}

.cctp-ledger th,
.cctp-ledger td {
  padding: 0.5rem 0.7rem;
  border: 1px solid var(--border, #e4e9f2);
  text-align: left;
}

.cctp-ledger th {
  background: var(--accent-softer, #f5f7ff);
  color: var(--heading, #1b2333);
}

.cctp-ledger td.num {
  text-align: right;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.cctp-scroll {
  overflow-x: auto;
}
</style>

## 事故速览

- 事发时间 <mark>2026-08-19</mark>，铺垫时间 <mark>2026-07-26</mark>，前后跨度约 24 天
- 涉及链：Polygon（伪造消息发出端）→ Base（Router 被打端）
- 直接损失 <mark>约 189,751 USDC</mark>（约 19 万美元）
- 攻击类型：[跨链消息校验缺失](/tags#CrossChainMessageFault) + 闪电贷凑余额
- 关键教训：**Circle 的 attestation 只证明「这条消息真的被发出过」，不证明「源链真的销毁了 USDC」**

> 注意区分：2026 年 7 月 19 日 Allbridge 还发生过另一起约 165 万美元的 Solana 侧闪电贷事故（同一个 Pool 账户被同时当作 send 和 receive 传入），那是完全不同的问题，本文不涉及。

## 背景：CCTP 到底证明了什么

Circle 的 CCTP（Cross-Chain Transfer Protocol）是「销毁—铸造」模型的跨链方案，正常流程是：

1. 用户在源链调用 Circle 的 `TokenMessengerV2.depositForBurn()`，合约**销毁**用户的 USDC；
2. `TokenMessengerV2` 内部再调用 `MessageTransmitterV2.sendMessage()`，把一条描述这笔转账的消息广播出去，产生 `MessageSent` 事件；
3. Circle 的链下 Attestation 服务看到这个事件，对消息体签名，产出 attestation；
4. 有人拿着 `(message, attestation)` 在目标链调用 `MessageTransmitterV2.receiveMessage()`，验签通过后回调 `TokenMessengerV2`，**铸造**等量 USDC 给接收方。

问题就藏在第 2 步和第 3 步之间的一个事实里：

<mark>MessageTransmitterV2.sendMessage() 是一个通用消息通道，任何地址都可以直接调用它。</mark>

也就是说，攻击者完全可以跳过第 1 步（不销毁任何 USDC），直接调 `sendMessage()` 发一条「长得像 CCTP 转账」的消息。Circle 的签名服务按标准流程照样会给它出 attestation——因为从 Circle 的角度看，这条消息**确实**由 `MessageTransmitterV2` 发出过，签名是对「消息存在性」的证明，不是对「资产已销毁」的证明。

真正把这两件事绑定在一起的，是消息头里的 `sender` 字段：

- 正常路径：消息由 `TokenMessengerV2` 代为发出，`sender` 等于源链官方 `TokenMessengerV2` 地址；
- 伪造路径：消息由攻击者自己发出，`sender` 等于攻击者的地址。

只要下游认真校验 `sender`，伪造消息就没有立足之地。Allbridge 恰恰漏掉了这一步。

## 攻击时间线

<ul class="cctp-timeline">
  <li>
    <span class="cctp-when">2026-07-26 · Polygon</span>
    <span class="cctp-what">埋雷：直接调用 MessageTransmitterV2.sendMessage()</span>
    攻击者构造一条 CCTP 格式的消息：source domain 为 Polygon，destination domain 为 Base，声明金额 1,000,000 USDC，<code>feeExecuted = 0</code>，<code>destinationCaller</code> 指向 Allbridge 的 <code>CCTPTokenMessenger</code>，接收方指向自己的攻击合约，并把预先算好的 <code>messageHash</code> 塞进 <code>hookData</code>。全程没有任何 USDC 被销毁。
  </li>
  <li>
    <span class="cctp-when">2026-07-26 起 · 链下</span>
    <span class="cctp-what">Circle 正常签发 attestation</span>
    对 Circle 来说这就是一条格式合法、确实由 MessageTransmitter 发出的消息，于是按标准流程出具了有效签名。攻击者拿到了一张「官方认证的空头支票」。
  </li>
  <li>
    <span class="cctp-when">约 24 天 · 等待</span>
    <span class="cctp-what">等 Router 余额够大</span>
    伪造的凭据是 100 万 USDC，但真正能提走的上限受 Router 实际持有的 USDC 约束。攻击者要么等大额入账，要么自己补齐差额——他选择了两者一起用。
  </li>
  <li>
    <span class="cctp-when">2026-08-19 · Base</span>
    <span class="cctp-what">在一笔真实入账后的第 6 秒发动</span>
    Allbridge 的 relayer 刚为一笔合法的 CCTP 存款向 Router 铸入约 191,156 USDC，攻击者在 6 秒后紧跟着发出攻击交易。
  </li>
  <li>
    <span class="cctp-when">2026-08-19 · Base</span>
    <span class="cctp-what">receiveCctpMessage：记账 100 万 USDC</span>
    攻击合约带着 7 月伪造的 message 与有效 attestation 调用 Allbridge 的 <code>receiveCctpMessage()</code>。由于缺少校验，合约认定这是一笔真实到账，为对应的 <code>messageHash</code> 写下一条 1,000,000 USDC 的可赎回记录。
  </li>
  <li>
    <span class="cctp-when">2026-08-19 · Base</span>
    <span class="cctp-what">Aave 闪电贷补齐差额</span>
    借入 808,844 USDC 打进 Router，把余额凑到刚好 1,000,000 USDC，让「账面记录」和「实际余额」在这一瞬间对得上。
  </li>
  <li>
    <span class="cctp-when">2026-08-19 · Base</span>
    <span class="cctp-what">receiveToken：提走 999,000 USDC</span>
    Router 的提现逻辑只检查 <code>CCTPTokenMessenger</code> 里这个 <code>messageHash</code> 对应的 <code>receivedTokenAmount</code> 是否大于零，校验通过，扣掉 0.1% 手续费后放行 999,000 USDC。
  </li>
  <li>
    <span class="cctp-when">2026-08-19 · Base</span>
    <span class="cctp-what">还贷离场</span>
    偿还本金 808,844 USDC 加约 404.42 USDC 手续费，净获利约 <mark>189,751.55 USDC</mark>。
  </li>
</ul>

一笔账算下来是这样的：

<div class="cctp-scroll">
<table class="cctp-ledger">
  <thead>
    <tr><th>步骤</th><th>Router USDC 余额变化</th><th>攻击者收支</th></tr>
  </thead>
  <tbody>
    <tr><td>relayer 合法铸币</td><td class="num">约 191,156</td><td class="num">—</td></tr>
    <tr><td>伪造消息记账</td><td class="num">不变</td><td class="num">账面 +1,000,000（凭空）</td></tr>
    <tr><td>Aave 闪电贷注入</td><td class="num">1,000,000</td><td class="num">−808,844（负债）</td></tr>
    <tr><td>receiveToken 提现</td><td class="num">约 1,000</td><td class="num">+999,000</td></tr>
    <tr><td>偿还闪电贷</td><td class="num">不变</td><td class="num">−808,844 − 404.42</td></tr>
    <tr><td><strong>净额</strong></td><td class="num">约 −190,156</td><td class="num"><strong>约 +189,751.55</strong></td></tr>
  </tbody>
</table>
</div>

Router 少掉的这 19 万，正是那笔属于真实用户的合法入账。

## 漏洞剖析：三道本该存在的关卡

根据 [SlowMist 的分析](https://slowmist.medium.com/a-cross-chain-attack-spanning-one-month-analysis-of-the-allbridge-hack-32a6183bce08)，Allbridge 在这条链路上连续漏掉了三处校验。下面是按公开分析还原的**简化伪代码**，用于说明控制流，并非逐字源码：

```solidity
// CCTPTokenMessenger（Base 侧）—— 攻击发生时的逻辑
function receiveCctpMessage(bytes calldata message, bytes calldata attestation) external {
    // 1) 让 Circle 的 MessageTransmitter 验签。这一步是真的，也确实通过了。
    //    但它证明的只是「这条消息由 MessageTransmitter 发出且被 Circle 签名」。
    IMessageTransmitterV2(messageTransmitter).receiveMessage(message, attestation);

    // 2) 从消息体里解析出金额、接收方、hookData 里的 messageHash
    (uint256 amount, address recipient, bytes32 messageHash) = _parse(message);

    // 缺失校验 A：message.header.sender 是否等于源链官方 TokenMessengerV2？
    // 缺失校验 B：message.body.recipient 是否等于本链 Circle TokenMessengerV2？
    // 缺失校验 C：Router 的 USDC 余额是否真的因为这次调用而增加了 amount？

    // 3) 直接落账：写下一条可赎回记录
    receivedTokenAmount[messageHash] = amount;   // 攻击者在这里拿到了 1,000,000 的凭据
}
```

```solidity
// Router（Base 侧）—— 提现侧
function receiveToken(bytes32 messageHash) external {
    uint256 credited = ICCTPTokenMessenger(cctpMessenger).receivedTokenAmount(messageHash);
    require(credited > 0, "no credit");          // 只验「有没有记录」，不验「钱是不是真到过」

    uint256 payout = credited - credited * fee / 1000;   // 0.1% 手续费
    IERC20(usdc).transfer(msg.sender, payout);   // 从 Router 的真实余额里付款
}
```

三处缺失可以这样理解：

| 校验 | 应该问的问题 | 缺失后的后果 |
| --- | --- | --- |
| A. `sender` | 这条消息是不是**官方 TokenMessenger 代发的**？ | 任何人直接调 `sendMessage()` 伪造的消息都被当成真转账 |
| B. `recipient` | 铸币的接收方是不是**本链官方 TokenMessengerV2**？ | 消息可以指向攻击者自己的合约，绕开真实铸币路径 |
| C. 余额 | Router 的 USDC **实际增加**了吗？ | 账面凭据与真实资产脱钩，凭据可以凭空产生 |

A 和 B 是**消息来源可信性**问题，C 是**账实一致性**问题。三者任意一个到位，这次攻击都不成立——这也是为什么它更像一次纵深防御全线失守，而不是单点疏忽。

## 两个细节：为什么等 24 天，为什么用闪电贷

这两个动作其实回答的是同一个问题：**伪造记账是免费的，但提现要花 Router 的真金白银。**

- 伪造凭据写的是 100 万，而 `receiveToken` 最终要从 Router 的真实 USDC 余额里转账。Router 平时余额远低于 100 万，凭据再大也提不出来。
- 于是攻击者用闪电贷把余额顶到 100 万。但闪电贷借来的钱最后要还，还款那部分是零和的——**真正的利润只能来自 Router 里原本就属于别人的钱**。
- 所以他必须等到 Router 余额尽可能高的时刻。8 月 19 日那笔 191,156 USDC 的合法入账就是他等的窗口，**6 秒**的反应速度说明这是自动化监控在盯着链上事件。

换个角度看：闪电贷在这里不是漏洞的一部分，而是**放大器**。真正的漏洞是凭据可以凭空创造；闪电贷只是让攻击者能把一张 100 万的假支票，兑现到 Router 余额允许的上限。

## 修复方向

SlowMist 给出的三条防线，正好对应上面三处缺失：

1. **验来源**：消息头的 `sender` 必须等于配置好的、源链上可信的 `TokenMessengerV2` 地址（按 `sourceDomain` 分别配置白名单）。
2. **验去向**：消息体里的 `recipient` 必须限定为本链 Circle 官方的 `TokenMessengerV2`，确保走的是真实铸币路径。
3. **验账实**：只有在 Router 的 USDC 余额**确实因本次铸币而增加**之后，才创建可赎回记录。实现上就是在调用前后各读一次 `balanceOf`，用差值而不是消息里声明的金额入账。

第 3 条尤其值得强调。它是最笨的一条，却也是唯一一条**不依赖你对上游协议语义理解正确**的兜底：哪怕对 CCTP 的消息格式有误解，只要以实际余额差为准记账，凭空凭据就不可能产生。

## 更一般的教训

这次事故可以浓缩成一句话：

> <mark>有效签名不等于有效资产。</mark>

跨链桥的本质是在目标链上复述一件发生在源链的事。签名回答的是「这句话是不是那边说的」，而不是「那边说的这件事是不是真的发生了」。当上游把一个通用消息通道（`sendMessage`）和一个资产通道（`depositForBurn`）复用同一套签名基础设施时，区分两者的责任就落到了集成方身上。

集成任何「消息 + 签名」式跨链协议时，值得逐条过一遍的清单：

- 这条消息的**发送者**是谁？是不是我信任的那个合约，而不只是「某个官方合约转发的」？
- 这条消息的**来源域**是不是我配置过的链？每条链的可信发送者是否分别配置？
- 消息里声明的**接收方 / 回调目标**是否被限定在白名单内？
- 消息声明的金额，和我这边**实际收到的资产**，是不是对得上？记账用的是哪一个？
- 同一条消息能否被**重放**？`messageHash` 的消费是否幂等？
- 这些校验在**单元测试里有没有对应的负例**？（伪造 sender、伪造 recipient、金额不匹配）

最后一条常常是分水岭。前面几条写下来都不难，难的是有人真的去构造一条「签名有效但资产不存在」的消息来跑一遍测试——而攻击者恰恰是最认真做这件事的人。

## 参考资料

- [SlowMist：A Cross-Chain Attack Spanning One Month, Analysis of the Allbridge Hack](https://slowmist.medium.com/a-cross-chain-attack-spanning-one-month-analysis-of-the-allbridge-hack-32a6183bce08)
- [SlowMist Hacked 区块链被黑档案库](https://hacked.slowmist.io/)
- [KuCoin：SlowMist 披露 Allbridge 跨链桥攻击细节](https://www.kucoin.com/news/flash/slow-mist-reveals-allbridge-cross-chain-bridge-attack-details-fake-cctp-messages-flash-loans-and-insufficient-minting-verification)
- [Allbridge Core EVM 合约仓库](https://github.com/allbridge-io/allbridge-core-evm-contracts)
- [Circle CCTP 官方文档](https://developers.circle.com/stablecoins/cctp-getting-started)
