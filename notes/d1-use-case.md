# 用例拆解卡：自动止损单

## 1. 监听哪条链、哪个合约的什么事件？
Ethereum Sepolia 上**某一个** Uniswap V2 交易对（TOKEN/WETH）的 `Sync` 事件。

订阅写法：`SYSTEM.subscribe(Sepolia 链 ID, 该交易对地址, Sync 的 topic0, REACTIVE_IGNORE, REACTIVE_IGNORE, REACTIVE_IGNORE)`。
- `contract_` 必须填具体的交易对地址。填 `address(0)` 会订阅整条链所有池子的 Sync，每条都要计费。
- 每次有人在这个池子里交易，池子都会发 Sync，里面带着最新的两种代币储备量。

## 2. 满足什么条件时触发？
订阅只能按“等于”过滤，没有“价格 < X”这种条件，所以条件要在 `react()` 里自己判断。

`react()` 的处理流程：
1. 先检查 `topic0` 和 `opCode`，再解码 `data`。（Sync 没有 indexed 参数，我认为 `opCode == 1`、两个储备量都在 `data` 里，部署前用真实日志核对。）
2. 用储备量算价格。要先确认 TOKEN 是 token0 还是 token1，决定用 reserve1/reserve0 还是反过来，并处理两个代币的 decimals。
3. 价格低于止损价，并且 `triggered == false` 时，先把 `triggered` 置为 true，再请求回调。

只触发一次的原因：
- Sync 每笔交易都会发，跌破后每个新 Sync 都会再触发一次。
- 止损卖出本身就发生在这个池子里，会产生新的 Sync，造成循环。
- 回调是异步的，所以 `triggered` 要在 `react()` 里立刻置位，不能等回调返回。
- 触发后可以在 `react()` 里 `unsubscribe`。六个参数要和 `subscribe` 完全一致，否则静默失败。
- 不匹配的日志直接跳过，不要 revert。revert 一样会被计费。

## 3. 在哪条链做什么动作？
回到 Ethereum Sepolia，由一个单独的回调合约把 TOKEN 卖成 WETH。

- Reactive 合约继承 `AbstractReactive`，`react()` 加 `onlySystem`，用 `SYSTEM.requestCallbackV_1_0(...)` 发请求。
- payload 的第一个参数传 `address(0)` 占位，网络会把它替换成 Reactive 合约地址。
- 回调合约继承 `AbstractCallback`，函数第一个参数是 `address`，并同时加 `onlyServiceProvider` 和 `onlyCallbackSender`。
- 回调合约要持有或被授权支配 TOKEN，swap 要设合理的 `minOut`。
- 一个合约不能同时是 Reactive 合约和回调合约。
- `gasLimit` 最低 100,000，低于这个值的请求会被忽略。proxy 还会先扣一部分，swap 的 gas 要算进去。
- 部署顺序：
  1. 部署 Reactive 合约。
  2. 部署回调合约，传入 callback proxy 地址和 Reactive 合约地址。
  3. 在 Reactive 合约上用 owner-only setter 设置 recipient。
  4. 两边充值。
- `callbackSender_` 构造时写死、没有 setter，填错只能重新部署。

## 4. 不用 Reactive 怎么做？Reactive 简单在哪？代价是什么？
不用 Reactive：自己写机器人跑在服务器上，7×24 小时盯价格，服务器上还要放钱包私钥来发交易。服务器挂了或网络断了，止损就失效。

Reactive 简单在哪：
- 不用自己维护服务器和轮询。
- 监听、判断、触发的规则都在合约里，合约源码经 Sourcify 验证后任何人都能核对。
- 私钥只在部署时用到。

代价：
- 费用公式是 `p_base · C · (g_used + K)`，K 默认 100,000 gas，可链上查询。每条 Sync 即使 `react()` 什么都不做，也要付一笔。
- Reactive 合约用 REACT 付费。测试网币名以官方页面为准，我读的三页没提 lREACT。
- 只要 debt 大于 0 就停止执行，不只是“余额不够”。
- 充值方式有两种：`depositTo()` 进 reserves，或者直接转账后调用自己暴露的 `coverDebt()`。
- 回调由回调合约在 Sepolia 用 ETH 付，失败的回调也照常收费。比如滑点导致 swap revert，钱照扣，而 `triggered` 已经置位，要提前想好失败后能不能重试。
- 池子越活跃，Sync 越多，花费越高。
- 订阅只能按“等于”过滤，复杂条件都要在 `react()` 里写。
- 单个 reactive 交易的 gas 上限是 900,000。

## 5. 简图
```
源链 Sepolia              Reactive Network               目标链 Sepolia
────────────              ────────────────               ──────────────
用户交易
   │
V2 交易对 ──emit Sync──▶  订阅过滤（交易对 + Sync）
                              │
                          react() [onlySystem]
                          算价格、比止损价、
                          查 triggered跌破止损价
                              │
                          requestCallbackV_1_0 ─────────▶ Callback proxy
                                                              │
                                                          止损合约（两道校验）
                                                              │
                                                          卖出 TOKEN → WETH
付费：—                   付费：REACT                    付费：Sepolia ETH
```