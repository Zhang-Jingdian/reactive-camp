# 读懂费用：用 cast 查询 Lasna 系统合约参数

## 命令与输出
```
export REACTIVE_RPC=https://lasna-rpc.rnk.dev/
export SYSTEM_CONTRACT_ADDR=0x8888888888888888888888888888888888888888

$ cast call $SYSTEM_CONTRACT_ADDR "_gasPriceCoeffPer1000()" --rpc-url $REACTIVE_RPC | cast to-dec
1000

$ cast call $SYSTEM_CONTRACT_ADDR "_extraGas()" --rpc-url $REACTIVE_RPC | cast to-dec
100000

$ cast call $SYSTEM_CONTRACT_ADDR "_maxChargeGas()" --rpc-url $REACTIVE_RPC | cast to-dec
50000
```
读数：C = 1000/1000 = 1，K = 100,000 gas，`pay()` 结算预算 = 50,000 gas。

## 1. 一次 react() 的费用怎么算？
fee = p_base × C × (g_used + K)

- p_base：`tx.gasprice` 和 `block.basefee` 中较大的那个。
- C：`_gasPriceCoeffPer1000 / 1000`。
- g_used：这次 `react()` 实际消耗的 gas。
- K：固定附加 gas，即 `_extraGas`。

`_maxChargeGas` 不进公式，它只限制系统合约调用 `pay()` 结算欠款时能用的 gas。

代入我查到的值：C = <输出1>/1000，K = <输出2>。假设某次 `react()` 消耗 g_used = 50,000 gas，则
fee = p_base × <C> × (50,000 + <K>)。

K 是固定加的，所以哪怕 `react()` 里什么都不做，每触发一次也要付至少 K 对应的钱。

## 2. balance、reserves、debt
- balance：合约自己账户里的钱，由合约自己控制。直接转账只增加 balance，既不产生 reserves，也不还 debt。
- reserves：预存在系统合约里的额度，只能用来付手续费。通过 `depositTo(合约地址)` 充入。
- debt：已经产生但还没付的手续费，记在系统合约里。

扣费时先花 reserves，reserves 不够才产生 debt。充值时先还 debt，还清了再增加 reserves，所以一个合约不会同时有 reserves 和 debt。

只要有一点 debt 就停止执行，是因为系统合约处理每个事件时，第一步就是检查 debt，在花任何 gas 之前。debt 大于 0，就不再调用 `react()`，网络也不再服务这个合约（会发出 `BlacklistContract` 事件）。因此合约是“安静地停下”，不是报错。debt 还清后会发出 `WhitelistContract`，服务恢复。

## 3. react() 回滚了还收费吗？
收，而且全额收。系统合约会捕获这次 revert，发出 `ReactiveContractReverted` 事件，然后照常按实际消耗的 gas 计费，不退款。

对合约设计的意义：
- 不要用 revert 当过滤器。不匹配的日志应该直接 `return`，不要 `revert`。
- 先检查 `topic0` 和 `opCode` 再解码 `data`，避免因为日志形状不对而 revert。
- 如果合约对每个事件都 revert，也会照样耗光资金，最后因为 debt 而停工。
- 回调侧同理：回调失败会发出 `CallbackFailure`，回调合约也要为这次尝试付费。