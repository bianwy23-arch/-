# 跨业务小模块故事与代码报告

> 目标：从十五条局部机制看懂团队如何处理并发、异步、资损、防重、异常收敛、性能瓶颈和稳定性降级。  
> 阅读方式：每章先读“四段故事”，再读“技术讲解”。故事中的“如果没有”是基于现有职责做的反事实推演，不代表故障曾在线上发生。

## 0. 先建立一张业务地图

这十五个模块分布在六类业务与技术治理场景里：

1. **红包**
   - Redis Lua 原子拆包：解决“多人同时抢”的并发分配。
   - 过期退款闭环：解决“没领完的钱如何回去”。
2. **BTC 卡消费**
   - PENDING 站内散兑：解决“用户冻结 BTC 如何变成待清算稳定币”。
3. **平仓**
   - 对冲结果同步状态桥：解决“交易所成交如何驱动内部结算”。
   - 退回金额三倍校验：解决“异常结果不能直接入账”。
4. **链上充币**
   - Soft Sibling 防重：解决“同一链上事实被不同 index 表达”。
5. **卡清算、拒绝与退款**
   - FAZZ 清算业务指纹：解决“渠道换幂等键重发同一清算”。
   - CAAS 拒绝原因标准化：解决“渠道拒因如何统一成文案和收费”。
   - 异常退款原交易匹配：解决“人工确认如何沉淀成自动匹配规则”。
6. **异常、性能与稳定性治理**
   - CAAS Advice 重试/DLT：解决“Advice 先于原授权单到达”。
   - 组合支付异步核销：解决“RPC 结果未知时如何安全重试”。
   - 对冲提交吞吐保护：解决“定时任务重入、漏扫和串行积压”。
   - 支付补偿游标分页：解决“状态变化导致 offset 漏扫及账户锁竞争”。
   - 稳定币脱锚状态机：解决“价格抖动、Redis 降级和告警盲区”。
   - 法币汇率多级降级：解决“外部渠道故障与无效重试风暴”。

它们并不是十五套相同的“加校验”。各自解决的是不同类型的不确定性：

- Lua 处理并发不确定性；
- 状态桥和 PENDING 散兑处理异步、乱序不确定性；
- Soft Sibling 与业务指纹处理外部标识不可靠；
- 三倍校验处理金额结果异常；
- 拒因映射处理渠道语义差异；
- 退款匹配处理业务证据不完整。
- RetryableTopic、DLT 与异步状态机处理外部结果未知；
- 游标、锁和批内串行处理规模及调度重入；
- 滞回、熔断和多级 fallback 处理基础设施与渠道故障。

---

# 第一章 红包 Redis Lua 原子拆包

## 1.1 四段故事

### 团队为什么要做

抢红包同时修改两种库存：剩余份数和剩余金额。单机锁只能保护一台应用实例，普通的“先读再写”也无法阻止两个请求同时读到最后一份。代码因此把查重、扣份数、算金额、扣金额和记录领取人放进同一段 Redis Lua 脚本，让 Redis 原子执行。

### 如果没有，流程哪里痛

这是反事实推演：

1. 红包只剩一份，两个请求同时读取 `surplusNum=1`；
2. 两个请求都判断“还能领”；
3. 两边都生成领取记录；
4. 后续数据库或资产池只能靠乐观锁拒绝其中一边，或者在防线不足时超发；
5. 用户会看到“抢到但不到账”，严重时红包总额不再守恒。

痛点并不只在性能，而是“谁抢到最后一份”无法形成唯一事实。

### 模块解决了什么

- `hexists` 保证同一 UID 对同一红包只分配一次；
- 在 Lua 内判断剩余份数并执行 `decrby`，避免超发；
- 均分模式最后一份拿走全部剩余；
- 随机模式先预留每份最小金额，再从剩余池随机分配；
- 返回 `-1 / 0 / 正金额`，让 Java 区分重复领取、抢光和成功。

### 对谁有影响

- **研发**：要同时理解 Redis 整数金额、Java `BigDecimal` 和数据库异步同步。
- **测试**：必须验证并发总额守恒，而不只是单线程接口成功。
- **产品**：重复领取、抢光、成功三种结果需要不同提示；现有外层异常处理会把重复领取弱化为金额 0。
- **财务/资产**：Lua 只决定“分到多少”，真正到账仍由后续资产入账决定。

## 1.2 `/teach` 技术讲解

### 最短调用链

```text
POST /api/v1/user/redpack/grabRedpack
→ UserRedPackController.grabRedpack
→ UserRedpackService.grabRedpack
→ UserRedpackCacheService.checkHasRedPack
→ UserRedpackCacheService.executeOpenRedPack
→ Redis EVAL UserRedpack.lua
→ saveRedpackRecord
→ executeSyncRedPackRecord
```

核心来源：

- [UserRedpackCacheService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/user/service/UserRedpackCacheService.java)：Redis Key、脚本调用和金额倍率还原。
- [UserRedpack.lua](../redotpay-api/redot-sys-service/src/main/resources/lua/UserRedpack.lua)：原子查重、分配和扣减。
- [UserRedpackService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/user/service/UserRedpackService.java)：领取编排、落领取单和异步同步。
- [UserRedpackValidatorService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/user/service/UserRedpackValidatorService.java)：不同币种的整数倍率。

### 关键设计

Redis 不直接保存小数。USDT/USDC 使用 `10000` 倍率，BTC/ETH 使用 `100000000` 倍率，其他币种按支持精度使用不同倍率。脚本返回整数，Java 再除以倍率恢复真实金额。

脚本返回值：

- `-1`：该 UID 已领取；
- `0`：红包不存在或已经抢光；
- 正整数：本次获得的倍率金额。

必须守住三个不变量：

1. 同一 UID 对同一红包最多成功一次；
2. 成功领取人数不超过总份数；
3. 所有已分配金额与最终剩余金额之和不超过红包总额。

### 代码边界与已知风险

Lua 原子性只覆盖 Redis。脚本成功后，`saveRedpackRecord` 仍可能失败，此时 Redis 已记录 UID，数据库却没有领取子单。后续同步任务看不到这次领取，形成 Redis 与数据库撕裂窗口。

### 测试现状

`UserRedpackCacheServiceTest` 中存在多线程拆包代码，但并发测试被注释；现有集成测试更接近手工驱动。缺少：

- `-1 / 0 / 正金额` 的脚本语义测试；
- 同 UID 并发重复领取；
- 不同币种倍率边界；
- Lua 成功、数据库落单失败的恢复测试。

### 排障练习

用户说“页面提示抢到了，但余额没有增加”。第一步应该只查 Redis 吗？

<details>
<summary>查看答案</summary>

不能。Redis 只能证明脚本分配过金额；还要沿 `user_redpack_record` 的同步、KYC 和资产入账状态继续检查。
</details>

---

# 第二章 红包过期退款闭环

## 2.1 四段故事

### 团队为什么要做

发红包时，资金已经离开发送人的可用余额，进入资产红包池。红包过期不是简单地把页面改成“已结束”，而是必须把资产池里尚未真正发给领取人的余额退回发送人，并让用户域同步展示退款结果。

### 如果没有，流程哪里痛

这是反事实推演：

1. 红包到期后不再允许领取；
2. 资产池仍保留 `surplusAmount`；
3. 发送人既不能继续发放，也拿不回余额；
4. 用户域主单一直是未结算，App 仍可能显示进行中；
5. 客服只能人工核查，财务账上出现长期悬挂资金。

如果只做资产退款而不做 Kafka 回执，则钱已经回来，但用户侧状态和通知仍是错的。

### 模块解决了什么

- 资产侧 Job 扫描过期且未结算的红包池；
- 以资产池 `surplusAmount` 作为退款真源；
- 退款入账与资产主单结算在事务内完成；
- 使用 `version + settleStatus + surplusAmount` 做乐观锁；
- Kafka 通知用户域写退款展示单、结算业务主单并通知发送人；
- 另一个任务关闭过期且未 KYC、未入账的领取子单。

### 对谁有影响

- **发送人**：决定未领余额能否及时回到基础账户。
- **领取人**：未 KYC 的领取意向在过期后会取消，不能再补入账。
- **产品/客服**：单人红包和多人红包有不同退款通知；双域状态不一致会直接产生客诉。
- **财务**：退款必须以资产池剩余为准，不能用 Redis 或用户域展示余额替代。

## 2.2 `/teach` 技术讲解

### 最短调用链

```text
assetUserRedpackExpireReturnJob
→ AssetUserRedpackService.executeRedpackExpireReturn
→ doExecuteRedpackExpireReturn
→ USER_REDPACK_RETURN 资产入账
→ AssetUserRedpackProducer.send
→ UserRedPackExpireReturnConsumer
→ UserRedpackService.processRedPackSettlement
```

核心来源：

- [AssetUserRedpackService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/asset/service/AssetUserRedpackService.java)：资产池扣减、过期退款和资产结算。
- [AssetUserRedpackJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/asset/AssetUserRedpackJob.java)：资产侧调度入口。
- [AssetUserRedpackProducer.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/mq/asset/AssetUserRedpackProducer.java)：退款完成事件。
- [UserRedPackExpireReturnConsumer.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/mq/UserRedPackExpireReturnConsumer.java)：用户域消费入口。
- [UserRedpackService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/user/service/UserRedpackService.java)：用户侧结算与未 KYC 子单关闭。

### 三种“剩余金额”不要混读

- Redis 剩余：抢包算法的整数库存；
- `user_redpack.surplus_amount`：业务域展示和编排数据；
- `asset_user_redpack.surplus_amount`：真正可退款的资产余额。

退款依据是第三种。资产侧先完成真实退款，再通过消息让用户域收敛。

必须守住的不变量：

```text
红包总额 = 已成功入账给领取人的金额 + 过期时资产池剩余
```

### 幂等与失败边界

资产 Job 只扫描未处理主单，并使用乐观锁避免重复退款。用户侧 `processRedPackSettlement` 看到已成功状态会直接返回，因此可承受重复消息。

需要注意：Consumer 捕获异常后仍结束消费。如果用户域并发更新失败，资产已经退款，但用户域可能永久不结算，除非另有补偿。

### 测试现状

现有测试覆盖领取入账和部分手工关闭调用，但未发现过期退款完整自动化测试。缺少：

- 资产池剩余退款及乐观锁冲突；
- Kafka 重复消费与消费异常；
- 未 KYC 子单关闭；
- “领取入账 + 剩余退款 = 原总额”的端到端守恒断言。

### 排障练习

发送人余额已经收到退款，但红包页面仍显示进行中，应该优先查哪一段？

<details>
<summary>查看答案</summary>

优先查资产退款事件、`UserRedPackExpireReturnConsumer` 和用户主单 `settleStatus`；资产余额已经证明资产侧完成。
</details>

---

# 第三章 BTC 消费 PENDING 站内散兑

## 3.1 四段故事

### 团队为什么要做

卡授权必须快速响应，因此授权阶段只冻结用户选中的 BTC，不等待交易所卖出。卡组织随后发送 PENDING 事件，系统才把冻结的 BTC 确认扣除、在用户账内换成稳定币并再次冻结，同时生成平台对冲单。

这使“用户支付账务”不必被“交易所成交速度”阻塞。

### 如果没有，流程哪里痛

这是反事实推演：

1. 卡授权成功，只留下冻结 BTC；
2. PENDING 到来后没有站内散兑；
3. `is_exchanged` 一直为 0；
4. 清算阶段拿不到已经冻结的稳定币，可能卡住或使用错误币种；
5. 平台没有对冲单，继续持有 BTC 价格敞口。

如果没有 CAS 防重，重复 PENDING 还可能重复确认冻结或重复散兑。

### 模块解决了什么

- 先创建交易所对冲待办；
- 确认 BTC 冻结，再完成 BTC→稳定币账内转换并冻结稳定币；
- 将 `payment_order.is_exchanged` 从 0 CAS 更新为 1；
- 已完成散兑的重复事件直接跳过；
- PENDING 丢失时，清算或撤销路径调用 `autoAuthorization` 补做散兑；
- 用户账务完成和平台交易所对冲异步解耦。

### 对谁有影响

- **用户**：决定 BTC 是否从“冻结”进入“已用于消费”的稳定币清算状态。
- **研发**：一次动作横跨支付单、闪兑单、资产账和对冲单。
- **测试**：必须覆盖 PENDING、DECLINED、CLEARED 的乱序和重复。
- **业务/财务**：站内散兑损益与交易所对冲损益属于两个阶段，不能混为一笔。

## 3.2 `/teach` 技术讲解

### 最短调用链

```text
PENDING webhook
→ CardTransactionWebhookV2Service.authorization
→ CardTransactionWebhookAuthorizationV2Service.authorization
→ tryCreateTransaction
→ confirmFreeze
→ transformAndThenFreeze
→ CAS is_exchanged = 1
```

清算先到时：

```text
CardTransactionWebhookClearingV2Service
→ autoAuthorization
→ authorization
```

核心来源：

- [CardTransactionWebhookAuthorizationV2Service.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/card/webhook/v2/CardTransactionWebhookAuthorizationV2Service.java)：PENDING 散兑和补偿入口。
- [CardTransactionWebhookClearingV2Service.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/card/webhook/v2/CardTransactionWebhookClearingV2Service.java)：清算前补散兑。
- [PaymentAutoExchangeOrderService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/v2/PaymentAutoExchangeOrderService.java)：闪兑单据。

### CAS 为什么是最后一道门

更新条件要求：

```text
status = PROCESSING
AND is_exchanged = 0
AND declined_status = UNPROCESSED
```

只有仍在处理、尚未散兑、也未被拒绝的支付单才能翻转状态。更新 0 行会抛并发更新异常，防止两个事件都宣称自己完成了散兑。

必须守住的不变量：

1. 授权阶段只冻结支付币；
2. PENDING 成功后支付币已确认、稳定币已冻结；
3. `is_exchanged=1` 才允许进入依赖散兑结果的清算；
4. 用户账内散兑完成不等于平台交易所对冲完成。

### 测试现状

已有测试覆盖欧洲 USDT→USDC 恢复和清算调用 `autoAuthorization`，但缺少：

- BTC PENDING 端到端资金变化；
- 重复 PENDING 的 CAS 行为；
- DECLINED 先到、PENDING 后到；
- `is_exchanged=1` 的幂等跳过；
- 清算、撤销和 PENDING 多事件竞争。

### 排障练习

交易所对冲还没成交，是否意味着用户的 BTC 一定还没完成站内散兑？

<details>
<summary>查看答案</summary>

不意味着。`is_exchanged=1` 表示用户账内散兑已完成；交易所订单是平台异步平敞口，两者刻意解耦。
</details>

---

# 第四章 平仓对冲结果同步状态桥

## 4.1 四段故事

### 团队为什么要做

平仓触发时，系统先接管质押物并创建交易所卖单。卖单由外部交易所异步执行，内部数据库事务不能一直等待。系统需要一个“状态桥”，把交易所最终结果翻译为内部平仓单可以继续结算的状态。

### 如果没有，流程哪里痛

这是反事实推演：

1. 质押物已经从用户质押账户转出；
2. 交易所实际已经完全成交；
3. 内部平仓单仍不知道结果；
4. 债务不核销，余款不退回；
5. 交易所账和内部账长期不一致，运营只能人工查单改状态。

另一种危险是跳过状态桥直接结算：尚未成交或只部分成交，却按预期金额还债和退款。

### 模块解决了什么

- 通过平仓单或子单 SN 查询 `ex_transaction_order`；
- 只有 `FILLED` 才同步实际成交数量和金额并进入成功状态；
- 提交失败或明确失败状态转为 FAIL，发送告警并等待人工审核；
- 尚未完成则保持 PROCESSING，下轮继续检查；
- 信用消费强平要求所有子单都成功，主单才可结算；
- 不需要交易所对冲的小额或稳定币路径直接写入预期处置结果。

### 对谁有影响

- **用户**：决定“平仓处理中”何时真正完成。
- **研发**：连接外部异步订单与内部强事务结算。
- **测试**：需要构造 FILLED、FAIL、部分成交、取消、重复轮询。
- **运营**：FAIL 状态和告警是人工审核入口。
- **财务**：实际成交金额是还债、收费和退余款的输入。

## 4.2 `/teach` 技术讲解

### 两条调用链

质押借币：

```text
LoanLiquidationJob.executeSettleLoanLiquidationJob
→ LoanLiquidationService.doSettleLoanLiquidation
→ validateBeforeSettle
→ processSyncExTransaction
→ liquidationCloseRepaymentSettle
```

信用消费强平：

```text
AssetUserCreditJob.creditUserClosingSettlementJob
→ AssetUserCreditCloseService.processCreditClosingSettlement
→ processSyncExTransaction
→ 全部子单成功门禁
→ 主单结算
```

核心来源：

- [LoanLiquidationService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/loan/service/LoanLiquidationService.java)：借币平仓状态桥和结算。
- [AssetUserCreditCloseService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/asset/service/AssetUserCreditCloseService.java)：信用消费强平子单状态桥。
- [TransactionOrderStatus.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/exchange/domain/lang/TransactionOrderStatus.java)：交易所失败状态集合。

### 状态门禁

借币平仓自动结算要求：

```text
orderStatus = PROCESSING
exStatus = PROCESSING
交易所 status = FILLED
```

同步成功后才把 `exStatus` 改成 SUCCESS，结算更新又要求 `exStatus=SUCCESS`。这是两次门禁，避免“同步函数返回错误”直接穿透到资金结算。

信用消费强平是子单级同步：任一子单失败或仍处理中，主单不能进入最终结算。

### 值得评审的边界

当前失败集合包含 REJECTED、EXPIRED、EXPIRED_IN_MATCH，但 CANCELED 和部分成交不一定进入 FAIL 分支，也不是 FILLED，可能长期停在 PROCESSING。报告只能确认代码行为，是否符合产品预期需要业务确认。

### 测试现状

现有 `LoanLiquidationJobTest` 更像手工集成入口，未对状态桥做自动断言。缺少：

- FILLED、提交失败、交易失败；
- 部分成交和取消；
- 无需对冲；
- 信用强平全部子单成功门禁；
- FAIL 后人工审核的前置条件。

### 排障练习

借币主单已经标记“平仓处理中”，为什么不能直接判断交易所卖单已经提交？

<details>
<summary>查看答案</summary>

“处理中”只说明平仓已启动。还要用平仓单 SN 查 `ex_transaction_order` 的提交状态与成交状态。
</details>

---

# 第五章 平仓退回金额三倍校验

## 5.1 四段故事

### 团队为什么要做

平仓最终会把“卖出所得减去欠款和费用”的余额退给用户。这个金额来自多个外部和内部数据：交易所累计成交额、债务快照、费用计算和人工审核输入。任何一项异常，都可能把不合理的大额资金写入用户账户。

代码在最终入账前设置了一条简单但明确的安全阀：退回金额不得大于欠款的三倍。

### 如果没有，流程哪里痛

这是反事实推演：

1. 交易所累计成交额或人工录入值异常偏大；
2. 结算公式算出异常高的退回金额；
3. 没有最后一道数量级检查；
4. 系统正常把超额资金打入用户账户；
5. 平台只能事后对账和追款，且没有即时告警。

### 模块解决了什么

- 在最终退款入账前比较 `returnAmount` 与 `totalAmount × 3`；
- 超限时发送 Lark 告警并抛出 `DevNeedCheckException`；
- 通过事务回滚阻止主单结算和资产退款；
- 信用消费强平对 `CLEAR_CREDIT` 合规清退用户提供豁免；
- 正常金额不额外查询合规记录，减少无关依赖。

### 对谁有影响

- **研发**：这是结算链路末端的资金安全阀，异常会回滚整个事务。
- **测试**：三倍边界、负退回、合规豁免和人工审核输入都应覆盖。
- **运营/风控**：超限不会静默失败，而是进入告警和人工核查。
- **用户**：异常单会延迟退款；合规清退用户可能合理持有高于欠款三倍的质押物。

## 5.2 `/teach` 技术讲解

### 两套公式

质押借币：

```text
returnAmount = 累计卖出所得 - 总欠款 - 预计平仓费
```

信用消费强平：

```text
returnAmount = 累计卖出所得 - 总欠款
```

调用位置：

- [LoanLiquidationService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/loan/service/LoanLiquidationService.java)：`calLiquidationFee` 与 `validateReturnAmount`。
- [AssetUserCreditCloseService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/asset/service/AssetUserCreditCloseService.java)：信用强平结算和合规豁免。
- [AssetUserCreditCloseServiceValidateReturnAmountTest.java](../redotpay-reap/redot-sys-service/src/test/java/com/redot/redotpay/business/asset/service/AssetUserCreditCloseServiceValidateReturnAmountTest.java)：信用侧现有边界测试。

### 规则的准确边界

- 只有 `returnAmount > 0` 且 `totalAmount > 0` 才进入三倍比较；
- 等于三倍不会被阻断，严格大于才阻断；
- 信用消费强平存在 `CLEAR_CREDIT` 豁免；
- 质押借币侧没有同等豁免；
- 质押借币负退回会归零并压缩费用，信用强平负退回直接报错。

这条规则证明“系统认为数量级异常时必须停下”，但源码不能证明为什么业务选择三倍而不是两倍或五倍。阈值依据需要产品、风控或历史事故资料确认。

### 必须守住的不变量

1. 超限结果不能更新平仓成功；
2. 超限结果不能进入用户基础账户；
3. 告警与阻断必须同时发生；
4. 豁免只能用于明确的合规清退场景。

### 测试现状

信用侧已有三类单测：超限阻断、合规清退放行、未超限不查合规。借币侧没有对应单测，还缺少：

- 等于三倍和略高于三倍；
- `returnAmount=0`；
- 负退回；
- 人工审核写入预期成交额后触发超限；
- 借币合规清退是否应豁免的业务确认。

### 排障练习

平仓对冲已 FILLED，但结算状态仍回到 PROCESSING，日志显示退回超限。为什么同步成功状态可能没有保留？

<details>
<summary>查看答案</summary>

借币侧同步状态和金额校验位于同一事务。三倍校验抛异常会回滚刚写入的成功状态，等待人工核查。
</details>

---

# 第六章 链上充币 Soft Sibling 防重

## 6.1 四段故事

### 团队为什么要做

同一笔链上转账可能同时被扫链任务和托管渠道回调发现。不同来源对 `vout_index` 的解释并不总是一致：一边可能给 0，另一边给 -1 或链上解析值。数据库精确唯一键会把它们视为两笔不同交易。

Soft Sibling 模块用“同用户、同商户、同币种、同 txHash、同地址、同金额”识别业务上的同胞订单，刻意忽略 index 差异。

### 如果没有，流程哪里痛

这是反事实推演：

1. 扫链以 `voutIndex=0` 创建充币单；
2. 渠道随后以另一个 index 回调；
3. 精确唯一键查询未命中；
4. 系统再创建一张充币单；
5. 同一链上事实可能被重复通知、重复入账。

如果只有 Soft 查询而没有锁，扫链和回调仍可能同时通过查询后各自插入。

### 模块解决了什么

- 通过总开关、币种白名单和灰度用户逐层控制风险；
- 在建单临界区获取 Redis 分布式锁；
- 锁内先查精确唯一键，再查忽略 index 的 Soft Sibling；
- 命中时复用已有订单，不调用建单函数；
- index 不同的命中会发送告警；
- 管理端强制补单可显式 `skipSoftSibling`，但仍保留精确唯一键。

### 对谁有影响

- **用户**：避免重复入账；锁失败时则暂缓入账，依赖上游重试。
- **研发**：要区分链上交易身份、渠道表达和数据库唯一键。
- **测试**：需要覆盖回调与扫链竞争，而不仅是 Service mock。
- **运营**：通过 Soft 命中告警识别渠道 index 漂移；强制补单是高风险例外。
- **财务**：同一链上事实只能形成一次有效入账。

## 6.2 `/teach` 技术讲解

### 最短调用链

```text
渠道回调或扫链
→ ChainDepositService / ChainDepositServiceV2 / ChainScanDepositService
→ DepositSoftSiblingDedupService.resolveOrCreate
→ Redis lock
→ 精确 UK 查询
→ Soft Sibling 查询
→ 复用已有单或执行 createFn
```

核心来源：

- [DepositSoftSiblingDedupService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/wallet/service/DepositSoftSiblingDedupService.java)：开关、锁和编排。
- [WalletDepositOrderService.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/wallet/service/WalletDepositOrderService.java)：精确查询与 Soft Sibling 查询。
- [ChainDepositServiceV2.java](../redotpay-api/redot-sys-service/src/main/java/com/redot/redotpay/business/wallet/service/ChainDepositServiceV2.java)：生产调用方之一。

### 两级身份

精确身份：

```text
uid + mid + varietyCode + txid + voutIndex
```

Soft 身份：

```text
uid + mid + coin + txid + address + chainAmount
```

Soft 查询在应用层使用 `BigDecimal.compareTo` 比较金额，避免数据库 decimal scale 不同导致漏匹配。

### 锁失败为什么选择“不建单”

锁等待失败或线程中断时，服务返回 `null` 并告警。调用方统一中止当前入账路径，依赖 webhook 或 MQ 重投。这是“宁可晚到账，不冒重复入账风险”的取舍。

必须守住的不变量：

1. 同一链上业务事实最多生成一张有效充币单；
2. Soft 命中不得执行 `createFn`；
3. 只有显式管理端补偿才能绕过 Soft；
4. 绕过 Soft 也不能绕过精确唯一键。

### 测试现状

单元测试已覆盖开关、白名单、灰度、锁失败、Soft 命中、精确命中和管理端跳过。缺少：

- 锁中断；
- 同 index Soft 命中不告警；
- 扫链 index 解析与 Soft 防重端到端；
- 真实并发下扫链与渠道回调只创建一张单；
- 调用方收到 `null` 后的重试闭环。

### 排障练习

用户充币未到账，日志显示获取 `deposit:create` 锁失败。是否应该立即人工强制补单？

<details>
<summary>查看答案</summary>

不能直接补。先确认上游是否会重投、是否已有精确单或 Soft Sibling；强制补单绕过 Soft，必须证明链上确实是另一笔资金。
</details>

---

# 第七章 FAZZ 清算业务指纹

## 7.1 四段故事

### 团队为什么要做

数据库唯一键通常使用 `transaction_id + idempotency_key`。但 Processor 重发同一业务清算时可能换一个新的幂等键，于是技术唯一键合法，业务事实却重复。团队需要另一层“业务身份”判断。

### 如果没有，流程哪里痛

这是反事实推演：

1. Processor 重发同一清算，使用新的 `idempotency_key`；
2. 数据库唯一键允许插入；
3. Job 把两行都发往清算 Consumer；
4. 下游可能重复扣款或重复结算；
5. 财务和客服只能在对账或用户投诉后发现。

### 模块解决了什么

- 从稳定业务字段拼接清算指纹；
- 刻意忽略 `idempotency_key`、处理状态和部分回写字段；
- 同一业务指纹出现多行时，Job 跳过相关行并告警；
- 同一 transaction id 但金额等关键字段不同，允许作为不同业务事实继续分发；
- `BASE_1_ADJUSTMENT` 不进入这套普通清算分发。

### 对谁有影响

- **持卡人**：降低重复清算和重复扣款风险。
- **研发**：新增字段时必须判断它是业务身份还是处理过程字段。
- **测试**：不能只测“相同对象”，要测幂等键变化、金额变化和字段回填。
- **运营/财务**：命中行保持未发送，等待人工确认；`skipCheck` 是高风险续跑入口。

## 7.2 `/teach` 技术讲解

### 最短调用链

```text
FazzClearingDispatchJob
→ FazzClearingDispatchService.dispatch
→ dispatchSuccess / dispatchManualAdjustment
→ FazzClearingDupChecker
→ FazzClearingFingerprint
→ 命中则告警并 skip
→ 未命中则 markKafkaSent 并发送 Kafka
```

核心来源：

- [FazzClearingFingerprint.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/fazz/clearing/FazzClearingFingerprint.java)：指纹字段选择。
- [FazzClearingDupChecker.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/fazz/clearing/FazzClearingDupChecker.java)：页内和关联行重复判断。
- [FazzClearingDispatchService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/fazz/clearing/FazzClearingDispatchService.java)：跳过、告警和发送。
- [FazzClearingFingerprintTest.java](../redotpay-reap/redot-sys-service/src/test/java/com/redot/redotpay/business/fazz/clearing/FazzClearingFingerprintTest.java)：字段取舍的回归证据。

### 为什么某些字段必须忽略

- `idempotency_key`：正是它可能变化，不能用来判断业务重复；
- `bill_amount`：可能在处理后回写，入站重复行与已处理行不一致；
- Success 的 `transaction_type`：可能由 Consumer 后补；
- `status / kafka_sent`：是系统处理状态，不是清算事实。

金额、币种、卡标识、渠道参考号等决定业务事实，通常必须进入指纹。字段过多会漏拦，字段过少会误杀合法清算。

### 适用范围

业务指纹覆盖 Job 分发路径。Webhook inline 直发不经过这层，主要依赖 `markKafkaSent` CAS 和数据库唯一键。报告不能把指纹描述为所有入口的全局防重。

必须守住的不变量：

1. 同业务事实即使换幂等键也不得重复分发；
2. 同 transaction id 的不同金额可以是合法多次清算；
3. 命中行不能标记为已发送；
4. `BASE_1_ADJUSTMENT` 必须走独立处理链。

### 测试现状

现有测试较完整，覆盖幂等键变化、金额变化、忽略回写字段、页内/跨页重复和人工 `skipCheck`。仍缺少：

- inline 直发绕过指纹的约束测试；
- BASE_1 不进入分发候选的分发级测试；
- 指纹字段变化导致合法放行的更多场景；
- 被长期跳过行的运营闭环。

### 排障练习

两行记录 `transaction_id` 相同但金额不同，业务指纹应该一律阻断吗？

<details>
<summary>查看答案</summary>

不应该。金额属于业务事实；金额不同会形成不同指纹，代码允许合法的多次清算。
</details>

---

# 第八章 CAAS 拒绝原因标准化

## 8.1 四段故事

### 团队为什么要做

CAAS 返回的是渠道 `AuthReason`，平台内部却需要统一决定两件事：用户看到什么拒绝文案，以及这次拒绝是否收手续费。直接使用渠道默认 ISO 响应码会丢失语义，例如系统繁忙、Ledger 故障、用户余额不足可能被压成相同或不准确的码。

### 如果没有，流程哪里痛

这是反事实推演：

1. 渠道返回 `SYSTEM_BUSY`、`MCC_BLOCKED` 或 `OUT_OF_ACCT_OTB`；
2. 平台只能使用粗粒度 ISO 码或统一未知码；
3. 用户看到错误原因；
4. 本应豁免的平台或渠道问题可能被收费；
5. 本应收费的用户原因也可能被错误豁免。

### 模块解决了什么

- 优先把 `AuthReason.name()` 映射成平台内部标准码；
- 兼容历史上误传入的 ISO 数字码；
- 未知值统一回退到 SDE；
- 内部标准码再映射为持卡人 DR 文案；
- DR 结果同时决定拒绝手续费是否豁免；
- CAAS 自定义拒因优先于渠道映射，支持更精确的业务例外。

### 对谁有影响

- **用户**：看到的拒绝原因和实际扣除的拒绝费。
- **产品/客服**：统一 DR 文案，减少跨渠道解释差异。
- **研发**：新增 `AuthReason` 必须同步维护映射表。
- **测试**：需要验证“原因→标准码→文案→收费”整条链，而非只测 Map。
- **财务**：拒绝手续费流水取决于豁免判断。

## 8.2 `/teach` 技术讲解

### 最短调用链

```text
DeclinedCardTransactionBill.calcAmount
→ CardTransactionBillProxyService.exemptFeeAmount
→ CaasDeclineCodeToStandardCode.getStandardDeclineReasonCode
→ StandardCodeToDeclineReasonShow.cover
→ DeclineReasonShowCardholder.isExemptFee
```

核心来源：

- [CaasDeclineCodeToStandardCode.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/third/common/webhook/CaasDeclineCodeToStandardCode.java)：渠道原因标准化。
- [AuthReason.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/third/caas/enums/AuthReason.java)：CAAS 原始原因。
- [StandardCodeToDeclineReasonShow.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/third/common/webhook/StandardCodeToDeclineReasonShow.java)：标准码到用户文案。
- [CardTransactionBillProxyService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/card/domain/bo/bill/CardTransactionBillProxyService.java)：拒绝费消费方。

### 四级解析顺序

1. 空值返回 null；
2. 尝试按 `AuthReason` 名称解析；
3. 尝试按内部标准码兼容 ISO 输入；
4. 未知值回退 SDE。

SDE 在展示层没有专用映射时落到 DR36，并豁免拒绝费。这是 fail-safe，但也意味着新增 AuthReason 如果漏配，可能静默变成“未知且不收费”。

测试中的 `everyAuthReasonHasMapping` 因此很重要：它把“维护静态表”从注释要求变成可执行约束。

必须守住的不变量：

1. 每个 AuthReason 都有明确内部映射；
2. 平台或渠道故障不能误收用户拒绝费；
3. 用户原因不能因为粗粒度回退而随意豁免；
4. 自定义拒因优先级高于普通渠道拒因。

### 测试现状

已有测试覆盖 AuthReason、ISO 兼容、未知回退、全枚举覆盖和部分文案/收费组合。缺少：

- `DeclinedCardTransactionBill` 真实金额端到端；
- ISO 数字码实际进入账单字段的生产路径；
- SDE、ORIG_NOT_FOUND 等完整展示链；
- `StandardCodeToDeclineReasonShow` 独立全量映射测试。

### 排障练习

CAAS 新增了一个 AuthReason，但开发只修改了枚举。最可能出现什么回归？

<details>
<summary>查看答案</summary>

全枚举映射测试应失败；若测试未运行，生产中该原因可能回退 SDE，导致 DR36 和拒绝费豁免。
</details>

---

# 第九章 异常退款原交易匹配

## 9.1 四段故事

### 团队为什么要做

退款通常需要关联原消费，但渠道数据可能缺少可靠原交易 ID。精确和宽松匹配失败后，运营只能人工找到正确清算单。团队把历史人工确认结果聚合成规则，再用当前退款和候选清算的交易证据做自动匹配。

### 如果没有，流程哪里痛

这是反事实推演：

1. 异常退款没有可靠原交易 ID；
2. 前四级匹配都失败；
3. 系统无法自动找到原清算；
4. 退款单滞留，运营逐单核对；
5. 用户到账变慢，人工确认量增长；
6. 历史确认经验无法复用，每次都从头判断。

比“匹配不上”更危险的是误匹配，因此模块在多候选并列时选择拒绝自动处理。

### 模块解决了什么

- 规则开关开启后，在前置匹配失败时进入 Step5；
- REAP 只允许 `abnormalType=7`，FAZZ 不受该限制；
- 有 merchantId 时按 ID 查规则，没有时按标准化商户名；
- 先用规则找候选，再用卡、金额窗口、时间、币种和商户证据验证真实清算；
- 多个候选都成立时比较历史 `hit_count`；
- 唯一最高才采纳，并列则拒绝自动关联；
- 人工确认记录由定时任务聚合回规则表，形成经验闭环。

### 对谁有影响

- **用户**：决定异常退款能否自动、及时到账。
- **运营/客服**：减少人工关联；歧义时仍保留人工确认。
- **研发**：匹配被拆成开关、资格、规则候选、交易验证和裁决。
- **测试**：需要覆盖 REAP/FAZZ、merchantId/商户名、0/1/多候选。
- **财务**：绑错原交易会把退款归到错误消费，影响对账与资金占用。

## 9.2 `/teach` 技术讲解

### 自动匹配链

```text
TxPaymentRefundAbnormalV2Service.getRelativeTransaction
→ Step1-4 精确或宽松匹配
→ TxRefundAbnormalRelativeRuleMatchService.match
→ RefundAbnormalRelativeRuleService.findCandidateRules
→ TxRefundAbnormalRelativeTransactionLookupService
→ resolveByTransactionEvidence
→ 成功返回原清算，歧义返回 empty
```

人工经验沉淀链：

```text
allowRefundWithRelativeTid
→ payment_refund_abnormal_relative_confirm
→ RefundAbnormalRelativeRuleGenerationJob
→ ADB 聚合 hit_count
→ batchUpsertRules
```

核心来源：

- [TxRefundAbnormalRelativeRuleMatchService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/tx/service/TxRefundAbnormalRelativeRuleMatchService.java)：开关、资格和匹配编排。
- [RefundAbnormalRelativeRuleService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/v2/refundabnormal/RefundAbnormalRelativeRuleService.java)：规则候选与多候选裁决。
- [TxRefundAbnormalRelativeTransactionLookupService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/tx/service/TxRefundAbnormalRelativeTransactionLookupService.java)：交易级候选和证据过滤。
- [TxPaymentRefundAbnormalV2Service.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/tx/service/TxPaymentRefundAbnormalV2Service.java)：Step5 在完整级联中的位置。

### 裁决规则

```text
没有交易证据命中 → 不关联
只有一个候选命中 → 自动关联
多个候选命中 → 比较 hit_count
唯一最高 → 自动关联
最高并列 → 不关联，交给人工
```

当前代码只使用 `hit_count`。旧的 `hit_ratio / key_total_count` 已废弃，不能在报告中继续描述为现行裁决依据。

有 merchantId 时只走 ID 路径，不再用商户名 OR 查询；没有 ID 时才标准化商户名。这个选择减少模糊匹配，但也意味着错误 merchantId 可能让本来可用的名称规则失效。

必须守住的不变量：

1. REAP 非 type=7 不得进入规则匹配；
2. 规则必须处于有效状态；
3. 规则命中不等于交易命中，仍要验证清算证据；
4. 多候选并列时宁可不退，也不能绑错原单；
5. 人工确认只能提高规则证据，不能绕过交易级验证。

### 架构边界

Tx 路径已接入 Step5；ADB 侧 `AdbPaymentRefundAbnormalV2Service` 未接入相同规则匹配。两条执行路径可能产生行为差异，这是已观察到的代码边界，不应直接断言为缺陷，需确认 ADB 的产品定位。

### 测试现状

已有单元测试覆盖：

- 开关关闭；
- REAP 非 type=7；
- merchantId 与标准化商户名；
- 0/1/多候选；
- `hit_count` 唯一最高、并列和 null；
- REAP/FAZZ 规则生成。

仍缺少：

- Step5 真实数据库规则和清算候选的集成测试；
- 多候选并列进入整条退款链路的端到端测试；
- merchantId 与 merchantName 同时存在时的优先级回归；
- 规则生成 Job 级测试；
- ADB 与 Tx 路径一致性验收。

### 排障练习

规则表查到两条候选，是否应该直接选择 `hit_count` 最大的一条？

<details>
<summary>查看答案</summary>

不能。先要让每条规则通过当前退款对应的交易级清算证据；只有多个真实候选都成立时，才用 `hit_count` 裁决，最高并列仍应拒绝。
</details>

---

# 第十章 CAAS Advice 原单等待与 DLT 分级兜底

## 10.1 四段故事

### 团队为什么要做

CAAS 的 0100 授权和 0120/0420 Advice 来自不同事件，Advice 可能先到，而它的 HOLD、NO_OP、REVERSAL 动作又依赖原授权单。更棘手的是 HTTP 已经向渠道返回 accepted，内部不能因为暂时找不到原单就把事件丢掉。

团队因此把“原单未就绪”拆成三个阶段：首次请求事务提交后投 Kafka、普通消费由 RetryableTopic 重试、重试耗尽进入 DLT，最后根据 `posting_mode` 做保守兜底。

### 如果没有，流程哪里痛

这是反事实推演：

1. HOLD Advice 比原授权单先到；
2. 系统查不到原单，却已经向渠道回复 accepted；
3. 如果直接丢弃，追加冻结永远不会发生；
4. 如果立即按无原单强制动账，又可能在原单稍后落库后重复冻结；
5. REVERSAL 若对不存在的原单强行释放，还可能释放错误资产。

痛点是渠道成功回执与内部账务完成不在同一时刻，必须有一个可收敛的异步承诺。

### 模块解决了什么

- 原单存在且已离开 PROCESSING 时，直接按 posting mode 处理；
- 首次未就绪时，在当前事务提交后投递 Kafka；
- 普通 Kafka 消费仍未就绪时抛 `ORIG_NOT_FOUND`，交给 RetryableTopic；
- DLT 后 HOLD 无原单才以 `auth_null` 强制建单冻结；
- DLT 后 REVERSAL 无原单选择跳过，避免对空单释放；
- 原单仍 PROCESSING 时始终告警并停止动账；
- Kafka 投递失败会独立告警，因为 HTTP 已 accepted，无法依靠渠道自然重试。

### 对谁有影响

- **用户**：HOLD 漏冻可能导致可用余额虚高，错误 REVERSAL 会造成资产释放异常。
- **研发**：必须区分 HTTP 接收成功、Kafka 投递成功和账务处理成功。
- **测试**：要覆盖事件乱序、重试耗尽及每种 posting mode 的 DLT 分支。
- **运营**：DLT 且原单仍处理中是人工核实入口。

## 10.2 `/teach` 技术讲解

### 最短调用链

```text
CAAS 0120/0420 Advice
→ AbsCaasWebhookHandler.adviceHandler
→ deferUntilOriginalOrderReady
→ 首次 afterCommit 投 caas-webhook-advice
→ CaasWebhookConsumer RetryableTopic
→ CaasWebhookAdviceRetryService.dlt
→ HOLD / NO_OP / REVERSAL 分级兜底
```

核心来源：

- [AbsCaasWebhookHandler.java](../redotpay-reap/redotpay-caas-base/src/main/java/com/redot/redotpay/third/caas/auth/AbsCaasWebhookHandler.java)：原单等待和 posting mode 分流。
- [CaasWebhookConsumer.java](../redotpay-reap/redotpay-caas-base/src/main/java/com/redot/redotpay/business/caas/consumer/CaasWebhookConsumer.java)：RetryableTopic 消费入口。
- [CaasWebhookAdviceRetryService.java](../redotpay-reap/redotpay-caas-base/src/main/java/com/redot/redotpay/business/caas/consumer/impl/CaasWebhookAdviceRetryService.java)：DLT 标记与回放。

### DLT 不是统一“强制执行”

进入 DLT 只表示自动等待已经耗尽，不表示任何动作都可以强做：

- `NO_OP + 无原单`：渠道已直接拒绝，跳过确认；
- `HOLD + 无原单`：创建订单并强制冻结；
- `REVERSAL + 无原单`：跳过撤销；
- `任意模式 + 原单仍 PROCESSING`：告警并停止。

必须守住的不变量：

1. 非 DLT 阶段找不到原单时不得直接强制冻结；
2. HOLD 最终不能静默丢失；
3. REVERSAL 不能对空单释放；
4. 原单 PROCESSING 时不能并发推进资产动作；
5. HTTP accepted 后，Kafka 投递失败必须可见。

### 测试现状

已有 `AbsCaasWebhookHandlerWebhookTest` 和 `CaasWebhookAdviceRetryServiceTest` 覆盖主要分支。仍缺少：

- RetryableTopic 真实重试到 DLT 的集成测试；
- 0100 与 0120 真实乱序；
- Kafka send 失败后的自动补投；
- DLT 强制冻结与 Card Core 资产变化的端到端断言。

### 排障练习

Advice 已经进入 DLT，原授权单仍是 PROCESSING，是否应该为了“最终一致”强制执行 HOLD？

<details>
<summary>查看答案</summary>

不应该。代码选择告警并停止，因为原授权事务仍可能完成；此时强制冻结会放大重复动账风险。
</details>

---

# 第十一章 组合支付异步批量核销

## 11.1 四段故事

### 团队为什么要做

组合支付事务提交后，需要调用 fx-core 批量核销多个报价。RPC 可能超时、连接重置、返回 5xx，甚至返回“部分 quote 已受理”的 `222011`。这些结果的共同点是：调用方无法简单判断“全部没处理”。

团队用 `exchange_async` 状态机保存请求和执行次数，在事务提交后异步调用，并让 Job 重试瞬态未知结果。

### 如果没有，流程哪里痛

这是反事实推演：

1. 支付事务已经提交，用户资金进入冻结或处理中；
2. fx-core RPC 超时；
3. 没有异步记录就无法重建原请求；
4. 盲目重试可能重复核销，直接失败则资金永久悬挂；
5. `222011` 若把整批 flash 记录都标失败，已核销项目未来可能被退款，形成用户双得。

### 模块解决了什么

- 事务内写入 INIT 和完整批量请求；
- afterCommit 后在线程池执行，避免阻塞提交和 HTTP 线程；
- INIT→PROCESSING 使用 CAS 抢占，防异步线程与 Job 双发 RPC；
- 超时、连接类异常和 HTTP 5xx 保持 PROCESSING；
- Job 扫描超时 INIT/PROCESSING，并限制最多执行三次；
- 正常成功后才把 async 和 flash 记录推进 SUCCESS；
- `222011` 只把 async 终止为 FAILED，flash 保持 PROCESSING，留给逐 quote 对账。

### 对谁有影响

- **用户**：决定组合支付冻结资金能否最终收敛。
- **研发**：必须按“结果未知”而非“调用失败”处理瞬态异常。
- **测试**：重点不只是成功/失败，还要覆盖半已知结果和并发抢占。
- **财务/运营**：`222011` 需要逐 quote 对账，不能整批自动退款。

## 11.2 `/teach` 技术讲解

### 最短调用链

```text
组合支付事务
→ ExchangeAsyncService.saveInitAndScheduleAsyncConfirm
→ afterCommit
→ executeBatchConfirm
→ INIT→PROCESSING CAS
→ fx-core batchConfirmTrade
→ SUCCESS / FAILED / 保持 PROCESSING

ExchangeAsyncRetryJob
→ retryTimeoutAsyncRecords
→ retryOne
→ executeBatchConfirm
```

核心来源：

- [ExchangeAsyncService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/ExchangeAsyncService.java)：状态机、RPC 和异常分类。
- [ExchangeAsyncRetryJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/payment/ExchangeAsyncRetryJob.java)：超时记录重试入口。
- [ExchangeAsyncServiceTest.java](../redotpay-reap/redot-sys-service/src/test/java/com/redot/redotpay/business/payment/service/ExchangeAsyncServiceTest.java)：瞬态异常与 `222011` 证据。

### 为什么 5xx 不能直接 FAILED

5xx 表示响应失败，不证明 fx-core 没有受理。代码已确认 fx-core 支持按业务标识幂等重试，因此选择保留 PROCESSING，让 Job 再试。非瞬态业务异常才推进 FAILED，并尝试标记 flash 记录失败。

`222011` 更特殊：它明确说明批次中已有项目被其他链路核销。此时停止整批重试，但不能把所有 flash 项目一起判失败。

必须守住的不变量：

1. 每次 RPC 前必须成功抢占状态；
2. 终态不可再次执行；
3. 瞬态未知结果不能落 FAILED；
4. `222011` 不能全量修改 flash 为 FAILED；
5. 只有顶层成功且结果条数一致时才整批成功。

### 测试现状

已有测试覆盖瞬态判断、503/超时保持 PROCESSING、成功和 `222011`。仍缺少：

- Retry Job 级集成测试；
- INIT 与 PROCESSING 并发抢占；
- 三次执行上限后的运营闭环；
- 请求反序列化失败后的冻结资金恢复；
- 组合支付到 fx-core 的端到端测试。

### 排障练习

RPC 返回 503 后，为什么不能立即把组合支付闪兑记录标成 FAILED？

<details>
<summary>查看答案</summary>

503 不能证明 fx-core 未受理；若实际已经核销，标失败后的退款或补偿会制造重复收益。代码保留 PROCESSING 并依赖幂等重试。
</details>

---

# 第十二章 对冲提交 Job 的吞吐保护

## 12.1 四段故事

### 团队为什么要做

卡消费散兑和平仓都会写待提交对冲单。定时任务每轮扫描后，通过同步 HTTP 逐笔送往交易渠道。如果上一轮尚未结束，新一轮又进入，就会重复扫描和放大外部请求；如果水位只向前推进，也可能越过尚未提交事务占用的较小 ID。

当前实现用进程内 `tryLock` 防重入，用 `lastId-200` 重叠窗口防漏扫，并在同一批内复用相同 symbol 的渠道选择结果。

### 如果没有，流程哪里痛

这是反事实推演：

1. 上一轮对冲 HTTP 尚未完成，下一次调度再次进入；
2. 多轮同时扫描同一批订单，外部请求和数据库写被放大；
3. 交易渠道更容易触发 429，平台敞口积压；
4. 若较小 ID 的插入事务未提交，水位被更大 ID 推进后可能永久漏扫；
5. 卡消费用户账已完成散兑，但平台仍暴露 BTC 价格风险。

### 模块解决了什么

- `tryLock` 失败时直接跳过，不让同 JVM 多轮排队；
- 扫描起点向前重叠 200 个 ID，补捞事务交错形成的空洞；
- 已处理订单依靠状态条件过滤，不会因重叠重复提交；
- 同一批、同一 symbol 只选择一次渠道；
- 成功 ID 与失败 ID共同参与水位更新，避免单个失败卡住全部进度。

### 对谁有影响

- **平台财务/风控**：提交速度决定未对冲敞口持续时间。
- **研发/SRE**：锁只能保护单 JVM，多实例仍需依赖数据库状态与渠道幂等。
- **测试**：需要证明重入被跳过、ID 空洞能补扫、失败不会卡死水位。
- **业务**：卡消费和平仓当前共享同一生产提交队列，会相互影响延迟。

## 12.2 `/teach` 技术讲解

### 最短调用链

```text
XXL submitTransactionOrder
→ ReentrantLock.tryLock
→ ScheduleHelperService.getLastId
→ scanFromId = lastId - 200
→ 查询 NOT_SUBMIT + PROCESSING + 未取消
→ for 循环串行 createOrder
→ ScheduleHelperService.setLastId
```

核心来源：

- [SubmitTransactionOrderJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/exchange/SubmitTransactionOrderJob.java)：锁、重叠扫描和串行提交。
- [ScheduleHelperService.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/schedule/ScheduleHelperService.java)：扫描水位。
- [SubmitTransactionOrderJob-bottleneck-CONCLUSION.md](../redotpay-reap/docs/test-reports/SubmitTransactionOrderJob-bottleneck-CONCLUSION.md)：仓库内瓶颈实验结论。

### 保护机制也是瓶颈来源

当前生产代码只有一把 JVM 锁，查询没有 SQL LIMIT，批内对外 HTTP 串行执行。因此积压 N 笔时，单轮耗时近似：

```text
单轮耗时 ≈ N × 单笔渠道 RTT
```

持锁越久，后续调度越容易 `tryLock` 失败整轮跳过。锁解决重入正确性，却不能提高吞吐。

仓库测试和设计文档出现卡消费/平仓分轨方案，但当前主类没有 `submitTransactionOrderCard`、`submitTransactionOrderLiquidation` 生产入口，不能把分轨写成已上线事实。

必须守住的不变量：

1. 同 JVM 同一时间只能有一轮生产提交；
2. 重叠扫描不能重复处理已提交订单；
3. 未提交事务形成的 ID 空洞必须能被后续补捞；
4. 外部提交结果未知时不能仅靠调度层盲目重发。

### 测试现状

瓶颈测试量化了串行耗时和锁跳过，`SubmitTransactionOrderJobLockSkipTest` 覆盖同入口重入。仍缺少：

- ID overlap 的事务交错回归；
- 多 schedule 实例并发；
- 真实 429 和渠道退避；
- 无 LIMIT 大积压的容量上界；
- 测试中分轨 API 与生产代码不一致的收敛。

### 排障练习

日志持续出现“检测到任务正在运行，自动跳过”，是否说明锁有问题？

<details>
<summary>查看答案</summary>

不一定。更可能是上一批串行 HTTP 耗时超过调度间隔。先查待提交数量、单笔 RTT 和单轮持锁时间，再判断是否需要限批或分轨。
</details>

---

# 第十三章 支付补偿的游标分页与 UID 隔离

## 13.1 四段故事

### 团队为什么要做

清算金额大于原授权冻结时，会产生支付补偿单。Job 处理成功后会把记录移出 PROCESSING 集合。如果使用 offset 分页，前一批状态变化会让后续数据向前移动，offset 继续增加时就会漏掉一部分记录。

同时，同一用户多笔补偿会竞争同一资产账户锁。代码因此采用 `id > lastId` 游标分页，并按 UID 分组：用户之间并行，同一用户内部串行。

### 如果没有，流程哪里痛

这是反事实推演：

1. 第一页补偿成功，状态从 PROCESSING 变为 SUCCESS；
2. PROCESSING 结果集缩短；
3. 第二页仍使用原 offset，跳过已经前移的记录；
4. 被跳过的 recover 单长期悬挂；
5. 如果简单改成全并发，同一 UID 多单会争抢账户行锁，增加等待和死锁概率。

### 模块解决了什么

- 按主键 ID 升序，每批最多 1000 条；
- 下一批使用 `id > lastId`，不受前批状态变化影响；
- 分近 30 天、30～90 天、90 天前三个任务窗口；
- 批内先按 UID 分组；
- UID 组之间在线程池并行，组内逐单串行；
- 余额不足时转入统一欠款台账，避免 recover 永久重试。

### 对谁有影响

- **用户**：决定补扣或欠款是否最终收敛。
- **研发**：分页正确性和并发形状同样重要。
- **DBA/SRE**：主键游标和固定批量限制单轮内存及查询压力。
- **风控**：补偿失败或转欠款会触发风险侧消息。

## 13.2 `/teach` 技术讲解

### 最短调用链

```text
paymentRecover 三档 XXL Job
→ recoverPagedById
→ id > lastId, ORDER BY id, LIMIT 1000
→ recoverInternal
→ groupingBy(uid)
→ 组间并行
→ 组内串行 tryRecover
→ 余额不足则 convertInsufficientRecoverToDebt
```

核心来源：

- [PaymentRecoverJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/payment/PaymentRecoverJob.java)：游标分页与并发隔离。
- [PaymentRecoverV2Service.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/payment/service/v2/PaymentRecoverV2Service.java)：单笔补偿、锁单和转欠款。

### 为什么游标不会因状态变化漏扫

游标只依赖稳定递增的主键，不依赖当前结果集位置。前一批即使全部变成 SUCCESS，下一批仍查询“大于已见最大 ID 的 PROCESSING 记录”。

UID 分组则把并发从“订单级”调整为“账户级”：

```text
不同 UID：可以并行
相同 UID：必须串行
```

必须守住的不变量：

1. 一轮扫描中每个满足条件的 ID 最多进入一次处理；
2. 前批状态变化不能导致后批漏扫；
3. 同一 UID 不能在本 Job 内并发扣同一账户；
4. 单笔失败不能中断整批；
5. 余额不足转欠款后不能继续无限直扣。

### 测试现状

`PaymentRecoverV2ServiceTest` 覆盖单笔业务逻辑，但未发现 Job 分页与并发专项测试。缺少：

- offset 与游标漏扫对照；
- 同 UID 组内串行的并发断言；
- 1000 条边界与多批处理；
- 三个时间窗口的边界；
- 转欠款后的下一轮不再直扣。

### 排障练习

为什么不能把 1000 条 recover 单直接全部扔进线程池并发执行？

<details>
<summary>查看答案</summary>

不同订单可能属于同一 UID，会竞争相同资产账户锁。按 UID 分组能保留跨用户吞吐，同时避免单用户内部锁争用。
</details>

---

# 第十四章 稳定币脱锚分级状态机

## 14.1 四段故事

### 团队为什么要做

稳定币价格会有正常微小波动，也可能出现真实脱锚。单阈值监控会在边界附近反复报警，而 Redis、告警通道或行情 tick 自身也可能故障。监控系统不仅要判断价格，还要保证“监控失效本身可见”。

团队建立 NORMAL、WARN、CRITICAL 三态滞回状态机：升 WARN 需要连续异常点，CRITICAL 可直接升级，恢复需要连续正常点，并坚持先发告警再提交状态。

### 如果没有，流程哪里痛

这是反事实推演：

1. 价格在阈值附近抖动；
2. 单阈值逻辑反复升降，告警淹没真正风险；
3. Redis 写失败时状态无法保存，每个 tick 都可能重复告警；
4. 如果先写状态后发告警，告警发送失败后系统却认为已经升档；
5. 后续同级状态不再告警，形成整个脱锚期间的永久盲区。

### 模块解决了什么

- 连续 WARN 点计数，正常点会清零，保证“连续”语义；
- CRITICAL 直接升档并清理历史计数；
- WARN/CRITICAL 恢复需要连续正常点，形成滞回；
- 先成功发送告警，再写入新状态；
- 告警成功但 Redis 写失败时允许下次重复告警，优先防漏报；
- Redis 读写或续期失败通过独立降级告警暴露；
- 跳过本轮价格检测时仍续期非 NORMAL 状态，避免 TTL 静默降级。

### 对谁有影响

- **风控/财务**：脱锚等级是暂停业务或人工响应的前置信号。
- **SRE**：能区分“价格正常”和“监控基础设施故障”。
- **研发**：要处理状态、计数器、TTL、告警限流之间的组合故障。
- **测试**：需要验证时间序列，而不是单次输入输出。

## 14.2 `/teach` 技术讲解

### 最短调用链

```text
StablecoinPegMonitorJob
→ 行情新鲜度与锚偏差检查
→ StablecoinPegStateMachine.onCheck
→ NORMAL / WARN / CRITICAL 分支
→ StablecoinPegCounter 连续计数
→ transitionTo
→ 先 trySendLarkAlarm
→ 后 writeState
```

核心来源：

- [StablecoinPegMonitorJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/market/StablecoinPegMonitorJob.java)：采样与检测编排。
- [StablecoinPegStateMachine.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/market/StablecoinPegStateMachine.java)：滞回和告警顺序。
- [StablecoinPegCounter.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/market/StablecoinPegCounter.java)：Lua 连续计数。
- [StablecoinPegDegradationAlarm.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/market/StablecoinPegDegradationAlarm.java)：Redis 降级可见性。

### 为什么“先告警，后写状态”

如果先写 CRITICAL，再发送告警，而告警发送失败，下一 tick 读取到的已经是 CRITICAL，只会走保持状态分支，不再重发升级告警。当前顺序让发送失败时状态保持原值，下一异常点还能重试。

告警限流只有在近期曾成功送达同级告警时才允许抑制，并且抑制时仍提交状态，避免状态永远无法推进。

必须守住的不变量：

1. WARN/CRITICAL 升档在首次成功提交前必须有告警送达；
2. 正常点会打断 WARN 连续计数；
3. 恢复必须满足连续点阈值；
4. 非 NORMAL 状态保持期间必须续期；
5. Redis 故障不得抛穿 Job，但必须产生降级信号。

### 测试现状

该模块测试较厚，包含 StateMachine、Counter、DegradationAlarm、TickStore 和 MonitorJob 等专项测试。仍缺少：

- WebSocket 行情写入到 Job 检测的端到端测试；
- 多 Pod 同时调度；
- Redis 写失败但读成功时的告警上界集成；
- 未来“CRITICAL 人工确认降档”接入后的行为测试。

### 排障练习

Redis 写状态失败后下一 tick 又发了一次 CRITICAL 告警，这是纯粹的重复告警缺陷吗？

<details>
<summary>查看答案</summary>

不完全是。当前设计有意选择“可能重复”而不是“状态已推进但告警永久漏报”，同时通过限流和降级告警控制副作用。
</details>

---

# 第十五章 法币汇率多级降级与双熔断

## 15.1 四段故事

### 团队为什么要做

法币汇率依赖多个外部渠道。主渠道可能超时、返回空值或陈旧价格；某些币种对则可能在所有渠道都没有报价。如果每次请求都从头打遍所有渠道，故障期间延迟、外部 QPS 和告警会同时放大。

团队建立缓存、主渠道、备用渠道、Floatrates 四级 fallback，并增加 Provider 级熔断和币种对级“空汇率熔断”。

### 如果没有，流程哪里痛

这是反事实推演：

1. 主汇率渠道发生抖动；
2. 所有业务请求同步等待其超时；
3. 重试继续打故障渠道，线程和连接逐渐耗尽；
4. 永远不存在报价的币种对每次都打遍三方渠道；
5. 告警和无效流量形成风暴；
6. 如果缓存里的汇率为 0 或陈旧，金额计算还可能得到错误结果。

### 模块解决了什么

- 缓存命中前统一校验数值和新鲜度；
- 主渠道失败后尝试备用渠道；
- 最后使用 Floatrates 兜底；
- Provider 熔断器阻止持续调用故障渠道；
- 所有渠道均无结果时，把币种对写入 Redis ZSET；
- 空汇率冷却期内快速拒绝，期满允许穿透一次尝试自愈；
- 不可报价币种前置拒绝，不污染“所有渠道失败”的熔断语义；
- 恢复成功后从空熔断集合移除。

### 对谁有影响

- **用户/产品**：消费汇率展示和多币种折算的可用性。
- **研发**：要区分无效值、陈旧值、渠道故障和不可报价币种。
- **SRE**：双熔断减少故障放大，恢复 Job 负责自愈。
- **财务**：零、负数或过期汇率不能进入金额链路。

## 15.2 `/teach` 技术讲解

### 最短调用链

```text
业务读取汇率
→ CurrencyExchangeRateRouter.fetchRateWithFallback
→ Tier1 Redis 缓存
→ 空汇率 ZSET 冷却判断
→ Tier2 主渠道 + Provider CircuitBreaker
→ Tier3 备用渠道
→ Tier4 Floatrates
→ 全失败则 markPairEmpty
→ CacheJob 周期重试并移除已恢复币种对
```

核心来源：

- [CurrencyExchangeRateRouter.java](../redotpay-reap/redot-sys-service/src/main/java/com/redot/redotpay/business/sys/service/cache/CurrencyExchangeRateRouter.java)：四级路由、数值校验和空熔断。
- [CurrencyExchangeRateRouterTest.java](../redotpay-reap/redot-sys-service/src/test/java/com/redot/redotpay/business/sys/service/cache/CurrencyExchangeRateRouterTest.java)：降级与熔断测试。
- [CacheJob.java](../redotpay-reap/redotpay-pay-schedule/src/main/java/com/redot/redotpay/jobs/sys/CacheJob.java)：空汇率恢复任务。

### 两类熔断解决不同问题

Provider 熔断器回答：

```text
这个外部渠道当前是否健康？
```

空汇率 ZSET 回答：

```text
这个具体币种对是否已经证明所有渠道都没有结果？
```

前者减少对故障服务的持续调用；后者避免对不可获得的币种对反复打遍全部渠道。不可报价币种属于确定性业务拒绝，不能写入空熔断，否则会混淆语义。

必须守住的不变量：

1. 汇率必须非空且大于 0；
2. 缓存必须通过新鲜度校验；
3. 空熔断只在所有渠道都失败后写入；
4. 冷却期满必须允许一次恢复探测；
5. 恢复成功必须清理空熔断状态。

### 测试现状

已有测试覆盖缓存、主备渠道、Provider 熔断和空汇率冷却/穿透。仍缺少：

- OANDA、NOWAPI、Floatrates 的真实联调；
- 冷却期满穿透成功并清理 ZSET 的端到端测试；
- Nacos 动态切换主备渠道与熔断状态联动；
- Redis 故障时整条汇率路由的明确降级策略。

### 排障练习

一个币种被标记为“不可报价”，是否应该加入空汇率熔断 ZSET？

<details>
<summary>查看答案</summary>

不应该。不可报价是确定性业务属性；空汇率 ZSET 表示可报价币种对已经尝试全部渠道仍失败，两者恢复方式不同。
</details>

---

# 十六、横向总结：十五个模块共同教会了什么

## 16.1 不确定性必须在正确边界被消化

- 并发争抢放在 Redis 原子脚本里解决；
- 外部异步成交通过状态桥进入内部强事务；
- 渠道标识不可靠时建立业务身份；
- 金额来源复杂时在入账前设置数量级安全阀；
- 渠道语义不同先标准化，再驱动文案和收费；
- 自动匹配证据不充分时选择停下，而不是猜一个答案。
- 事件乱序通过 RetryableTopic 与 DLT 分阶段收敛；
- RPC 未知结果通过持久状态、CAS 和幂等重试消化；
- 大表扫描用游标分页，账户并发按 UID 隔离；
- 高危监控坚持先告警再提交状态；
- 外部渠道故障通过 fallback 与双熔断限制影响范围。

## 16.2 “没有模块”不等于“系统一定立刻资损”

数据库唯一键、乐观锁、下游 CAS 和人工流程有时仍会兜底。但如果删除这些模块，问题会被推迟到更晚、更昂贵、更难解释的位置：

- Lua 被删除后，冲突被推给数据库和用户体验；
- 指纹被删除后，重复事实被推给清算 Consumer；
- 状态桥被删除后，交易所事实无法进入账务；
- 退款规则被删除后，判断成本重新回到运营。
- 异步核销被删除后，RPC 未知结果只能靠人工对账；
- 游标和 UID 隔离被删除后，补偿会漏扫或制造账户锁竞争；
- 熔断与滞回被删除后，渠道故障和价格抖动会被放大成系统故障。

模块价值常常不是“唯一能防住”，而是“在最便宜、最懂业务的边界防住”。

## 16.3 评审这类模块的八个问题

1. 它认定的业务唯一身份是什么？
2. 它读取的是展示数据、缓存数据，还是资金真源？
3. 并发或重复事件到来时，第二个请求在哪里被拒绝？
4. 外部系统结果不完整、乱序或未知时，是重试、等待、告警还是人工？
5. 模块失败后，是晚到账、卡单、误收费，还是可能直接资损？
6. 测试证明的是单个函数，还是完整链路的不变量？
7. 数据量、外部 RTT 或调度频率翻倍后，复杂度和持锁时间如何变化？
8. 辅助组件或告警通道故障时，模块是静默、降级还是有独立信号？

## 16.4 报告结论的证据等级

- **源码事实**：本报告通过实现、调用方、DAO 和测试直接确认。
- **反事实推演**：删除模块后沿现有链路推导的可能故障，不代表真实事故。
- **待确认项**：三倍阈值依据、部分成交长期 PROCESSING 是否符合产品预期、ADB 未接 Step5 的定位、对冲分轨是否计划落地、生产 Job 与开关配置，需要领域负责人或生产配置补证。

---

# 十七、建议的复习顺序

第一次复习按风险从直观到抽象：

1. 红包 Lua：看懂原子性；
2. 红包退款：看懂资金真源；
3. Soft Sibling：看懂业务身份；
4. FAZZ 指纹：看懂字段取舍；
5. BTC PENDING：看懂事件乱序与账务解耦；
6. 平仓状态桥：看懂外部状态如何进入内部结算；
7. 三倍校验：看懂资金安全阀；
8. CAAS 拒因：看懂语义标准化；
9. 异常退款匹配：看懂人工经验如何变成谨慎自动化。
10. CAAS Advice：看懂乱序事件和 DLT 分级兜底；
11. 组合支付异步核销：看懂 RPC 未知结果；
12. 对冲提交 Job：看懂正确性保护如何成为吞吐瓶颈；
13. 支付补偿分页：看懂游标与账户级并发；
14. 稳定币脱锚：看懂滞回和监控降级；
15. 法币汇率路由：看懂 fallback 与双熔断。

如果某一章仍不清楚，可以继续问我三个固定问题：

- “这一步之前，钱或状态在哪里？”
- “这一步之后，哪个字段成为下游门禁？”
- “重复、失败或乱序时，第二条防线是什么？”
