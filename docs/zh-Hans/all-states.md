# 全部状态（一张表）

[返回目录](README.md)

## A · 链上与 Ark 资金

### [A1 转入 Ark、直接发送 BTC、RBF 加速](ark-onchain/A1-onboard-direct-send-rbf.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | 报价生成即落库，有效期 300 秒 | 未签名 PSBT；资金仍在链上钱包，未加锁 | `wallet.bitcoin.review` | 确认；或放弃报价（加速报价可删除） | 否（无过期清理，过期后确认被拒） | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | 已签名并写入 `signed_tx_hex` | 签名交易在本地；直接发送已 `register_tx` 进 BDK，余额按已花计；onboard 输入被锁定 | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | 只能等待 | 否 | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | 链源报 Mempool | 交易在内存池 | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | 等待；直接发送可 RBF | 否 | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | 仅 onboard：资金交易已确认，Ark 尚未登记 | 资金在 board 输出；board VTXO 被 movement 锁定 | 同上 + `wallet.bitcoin.waiting-ark` | 等待 | 否 | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | 链源查询失败、看不到交易、onboard 未拿到联签或 movement 缺失 | 直接发送：签名交易仍在 BDK；onboard：输入锁定 | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | 等待 | 否 | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | 直接发送：某次尝试被确认；onboard：movement 成功且有退出凭据 | 直接发送：已到对方；onboard：在 Ark | `wallet.bitcoin.completed` | — | 是（直接发送遇链重组可回到 Unknown） | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | 仅 onboard：服务器条款变化（`submit_signed_board` 返回 None），或 Bark movement 记为 Failed/Canceled | 条款变化时：输入仍锁定；movement 失败时：资金可能处于退出流程中 | `wallet.bitcoin.failed`（"Transfer failed. Review a new quote to try again."） | 重新报价 | 是 | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | 拿到联签、VTXO 已构建 | 资金交易待推送；VTXO 以 Locked 存库 | — | 见到上链 → `Confirming`；输入被确认交易冲突 → 失败；否则停泊等待 | 否 | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | 资金交易在内存池或区块 | board VTXO Locked，计入 `pending_board` | — | 确认数够即登记；登记失败则重试；临近到期自动发起退出 | 否 | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed`（退出后） | VTXO 已 Exited 或已被冲突花费 | 前者在退出流程中，后者资金未离开 | — | 终态 | 是 | `BK/actions/board.rs:299-314` |
| A1-11 | 加速报价 `Reviewing` + `replacement` | 用户请求加速，条件满足 | 未签名替换 PSBT，300 秒有效 | — | 确认；或取消（只删未签名记录） | 否（随谱系） | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | 父记录 `replaced_by` 置位、子记录签名 | 用户确认 | 父子同一事务提交 | — | 等待 | 否（随谱系） | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | 谱系状态 | 任一尝试确认即该尝试 Completed，其余维持；头部按内存池/未见判定 | — | — | — | 否（随谱系） | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 从 Ark 发送 BTC 到链上地址](ark-onchain/A2-ark-to-onchain-send.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing`（Bark `Start` / `SplitWithArkoor`） | 用户确认，动作检查点写入 | 输入 VTXO 被动作锁定 | `wallet.bitcoin-send.preparing`（"Payment in progress"） | 「检查状态」 | 否 | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing`（Bark `ArkoorRegistrationRequired`） | 已做 arkoor 拆分，原输入标为已花 | 拆出的发送 VTXO **与找零** 都被动作锁定，等待向服务器登记 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing`（Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`） | 登记成功，与服务器交换 offboard 交易与 forfeit | 发送 VTXO 锁定；找零已解锁 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast`（Bark `AwaitingConfirmations`） | offboard 交易已推送 | 资金在链上交易里；VTXO 已 forfeit | `wallet.bitcoin-send.broadcast` | 「检查状态」 | 否 | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | movement Successful | 已到对方地址 | `wallet.bitcoin-send.confirmed` | — | 是 | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | movement Failed / Canceled（Bark 在 `Start`、`SplitWithArkoor`、未经 forfeit 的 `ReadyForOffboard` 失败时释放输入） | 输入回到可花 | `wallet.bitcoin-send.failed` | 重新报价 | 是 | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | 结果未知（确认调用超时或报错） | `start_send_onchain_confirmed` 已写检查点但返回超时 | 同所在 Bark 状态 | `wallet.bitcoin-send.uncertain` | 先查原操作，不能新建 | 否 | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 出口预留](ark-onchain/A3-exit-fee-reservation.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`，`remaining_sats = Some(0)` | 新账户登记；没有未认领的退出 | 无预留 | 无 | — | 否（可再次进入 Active） | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`，`Some(n)` | 有进行中的退出，且能估算出手续费 | 共享 BTC 中 n sat 被视为预留，从可用余额扣除 | 余额结构里有 `reserved_sats` / `available_sats`（`W/ark/types.rs:429-439`） | 用户发送时如侵占预留须单独确认（`BitcoinExitReserveImpact`） | 否 | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`，`Some(上次值)` | 估算失败，保留上次可信值 | 同上 | 同上 | 同上 | 否 | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`，`None` | 恢复开始、恢复出新退出、从 Idle 首次发现退出、或授权新退出但不可重估 | 预留未知，同网络的直接发送与 onboard 暂不可用 | 发送报错 | 无 | 否 | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | 待登记的恢复退出（`pending_recovered_exit_ids` 非空） | 恢复出的退出已登记、Bark 检查点未齐 | 同上 | 同上 | 无 | 否 | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 强制退出与认领](ark-onchain/A4-force-exit-and-claim.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | 用户确认强退，或系统为 board / HTLC 自动退出 | 退出链交易逐层广播，需要共享 BTC 付 CPFP | `wallet.ark.exit-in-progress`、`wallet.ark.pending-funding`、`wallet.ark.refresh-to-progress-exit` | 刷新推进；补充链上资金 | 否 | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | 全部退出交易确认 | 退出输出，时间锁未到 | `wallet.ark.claim-timelock-running` | 等待 | 否 | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | 时间锁到期 | 退出输出，仅用户可花 | `wallet.ark.claim-to-bitcoin-address` | 认领 | 否 | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | 链上出现花费该输出的内存池交易 | 认领交易在内存池 | `wallet.ark.claim-in-progress` | 等待 | 否；交易掉出内存池回到 `Claimable` | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | 花费交易确认 | 在用户地址 | — | — | 是 | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO 已在链下被花；或服务器在退出中途清扫 | 无 | — | — | 是 | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | 最终交易广播前取消 | VTXO 仍可花 | — | 重新发起 | 可恢复 | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | 用户确认报价，签名交易落库，输入登记到 `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | 「提交原交易」 | 否 | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | 开始广播（先改状态再广播） | — | 认领中 | 等待 | 否 | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | 广播返回成功，或链源报内存池 | — | `wallet.ark.claim-in-progress-message` | 「刷新」 | 否 | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired`（投影） | 链上已确认但写库失败 | — | `wallet.ark.claim-confirmation-checkpoint-required` | 刷新 | 否 | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | 链上确认并写库 | — | 已认领 | — | 是 | `claim_store.rs:287-293` |

### [A5 刷新与自动维护](ark-onchain/A5-refresh-and-maintenance.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | 计划刷新已被服务器接受，等待目标高度；计划等待：是（`is_scheduled_waiting`） | 输入未锁，仍可强退 | — | — | 否 | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | 委托刷新已提交，等待服务器出轮次；计划等待：否 | 输入被 movement 锁定，计入 `pending_in_round` | — | — | 否 | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | 交互式轮次排队 / 进行中；计划等待：否 | 同上 | — | — | 否 | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished`（未 done） | 交互结束，等待轮次资金交易确认；计划等待：否 | 同上；forfeit 尚未完成 | — | — | 否 | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished`（done）→ `Confirmed` | 资金交易确认、forfeit 完成；计划等待：— | 新 VTXO | — | — | 是 | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | 轮次失败、用户取消、输入丢失无法缩减；计划等待：— | 输入释放 | — | — | 是 | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | 无需刷新 / 自动刷新关闭 | — | `disabled-title` | 开启或手动刷新 | 否（常驻） | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | 计划已批准 / 已提交 | — | — | 等待 | 否 | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | 失败退避 60s→30min | — | — | 等待 | 否 | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | 提交结果不明，需先恢复 | — | — | 等待（每 `URGENT_CHECK_SECONDS` 再查） | 否 | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | 需用户授权费用、人工复核、补资金 | — | `authorization-*`、`funds-title` 等 | 授权、复核或补资金 | 否（待用户） | `W/ark/maintenance.rs:35-62` |

## B · 支付与收款

### [B1 Lightning 收发（Ark）](payments/B1-lightning-ark.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | 用户确认支付 | 输入 VTXO 被动作锁定 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | 服务器联签 HTLC 输出 | HTLC VTXO 锁定，计入 `pending_lightning_send` | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | 已通知服务器付款 | 同上 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | 服务器报失败，或 HTLC 临近到期仍 Pending | 同上，准备撤回 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | 服务器拒绝撤回 | 同上 | `InProgress{has_failed_revocation: true}` | 等待 | 否 | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | 拿到原像 | 已付出 | `Paid` | — | 是 | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | 撤回完成（Done） | 服务器联签撤回 | 回到可花 VTXO | — | — | 是 | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | 生成发票 | 无 | — | — | 否 | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | 服务器准备好 HTLC-recv VTXO | 锁定，尚可放弃 | — | — | 否；认领窗口关闭或被拒即放弃 | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | 已向服务器交出原像 | HTLC-recv VTXO 锁定，计入 `claimable_lightning_receive` | — | — | 否；认领失败后停泊 | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | 认领输出需转交第三方 | 锁定 | — | — | 否 | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | 认领完成 | 新 VTXO | — | — | 是 | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Cashu 收款](payments/B2-cashu-receive.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | 收到 token，mint 已注册 | bearer token 存于本地加密库（唯一可重放副本） | 收款处理中 | 等待 | 否 | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked`（`UnknownMint`） | mint 未注册 | 同上 | 需添加 mint | 添加 mint 后继续 | 否 | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | 正在对账，或 SDK 已有该操作的 saga | token + CDK saga | 处理中 | 等待 | 否 | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue`（`RetryRequired`） | 对账出错且没有 SDK 操作 ID，30 秒后再试 | token | 处理中 | 等待 | 否 | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal`（issue 为空） | 收到 `Credited` / `AlreadyReceived` / `Recovered` | 已入 CDK 余额 | 已到账 | — | 是 | `receive_operations.rs:196-206` |
| B2-06 | `Terminal`（`AlreadySpent`） | mint 报 token 已被花 | 无 | `CashuTokenAlreadySpent` 报错 | — | 是 | `receive_operations.rs:207-213` |

### [B3 聊天转账、红包与收款回执](payments/B3-chat-transfer-red-packet.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`，无 provider 操作 | 用户确认金额 | 未动资金 | — | 继续 | 否 | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`，有 provider 操作（Cashu） | Cashu 发送 saga 已开始 | mint 侧 saga；token 可能已生成 | `wallet.chat.previous-cashu-interrupted` | 「继续」：重新从 saga 找回 token | 否 | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`，Ark 未决 | Ark 转账结果未知 | 可能已离开账户 | `wallet.chat.previous-wallet-transfer-interrupted` | 输入交易 ID 对账；或确认未发出后放弃 | 否 | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | 拿到 token（Cashu）或交易 ID（Ark） | 资金已离开钱包，token 在本地 saga | `wallet.chat.wallet-transfer-complete`（可续交消息） | 继续把消息发出 | 否 | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | 删除（完成） | 消息交给发送器 | 交接记录删除 | — | — | 是 | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | 无 → `Pending` / `Available` | 收到或发出 | — | 可领取 | 收款方领取；发送方（红包）不可领 | 否 | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | 收款方点领取，先落阶段再赎回 | — | 领取中，`can_retry` | 重试 | 否 | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | 赎回成功并发回执；或收到对方回执 | — | 已领取 | 回执失败可重发 | 是 | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | 发送方在红包过期后开始退回 | — | `chat.payment.shared.refunding` / `continue-refund` | 继续退回（界面自动续做） | 否 | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | 退回完成 | — | 已退回 | 回执失败可重发 | 是 | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed`（发送方观察） | 发送方发现 token 已被花但无回执 | — | `payment_spent_without_receipt` | — | 是（投影） | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 消息邮资与找回](payments/B4-relay-postage-reclaim.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | relay 不收邮资 | — | — | — | 是（该投递） | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | 尚未付或无法付 | 未付出 | — | — | 否 | same as above |
| B4-03 | `Created` | 已生成邮资 token 并随事件发给 relay | token 交给 relay，钱包流水记 `relay_stamp` 发送 | — | — | 是（该投递） | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | 不可找回（无绑定） | 投递行状态不在 failed / timeout / no_ack / terminal_failed / retry_exhausted 之列，或曾收到 OK | — | 不显示入口 | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | 绑定唯一对应一笔 `TokenCreated` 的发送 saga | token 仍可能未被 relay 兑现 | 显示找回 | 找回 | 否 | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | 已接受找回意图（先落库再动 mint），或 mint 状态不明、连不上 | 意图 + 原证明 | 处理中 | 再次找回 | 否 | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | 交换完成 | 回到钱包 | 已找回 | — | 是 | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | 证明全部已被花（relay 已兑现），或找不到唯一对应流水 | relay 处 | 不可找回 | — | 是 | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 付费通话票](payments/B5-paid-call-tickets.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | 用户确认购买，订单行写入 | 尚未出 token；或已出 token 尚未写入 | 付款中 | 继续 / 取消 | 否 | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | token 写入订单并交给 notify | token 在订单 | 确认中；确认页只给 Continue | 继续轮询 | 否 | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify 确认；随后按使用情况投影 | 通行证 | 可用 / 使用中 / 已用 | 拨打 | 是（支付面） | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify 回失败，或钱包出 token 失败 | token 被置空 | 失败 | — | 是 | `rtc_tickets.rs:118-125`, `:152` |

## C · 消息与关系

### [C1 消息发送](messaging/C1-message-send.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | 有链接预览意图未完成 | 发送作业 | 发送中 | 文本 / 文件可撤销 | 否 | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | 附件意图未完成 | 发送作业 + 本地附件 | 上传中 | 同上 | 否 | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | 作业存在，尚未交给传输层 | 发送作业 | 发送中 | 同上 | 否 | `delivery_queries.rs:430-432` |
| C1-04 | `Failed`（作业内） | 上传被暂停 | 发送作业 | 失败 | 重试 | 否 | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | 已生成传输事件，未达业务成功也未终态 | `app_message_transport_events` + 每 relay 投递行 | 发送中 | 等待 | 否 | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | 主要单元达到业务成功 | 投递行 | 已发送 | — | 是 | `delivery_queries.rs:433-437` |
| C1-07 | `Failed`（投递） | 传输策略判终态 | 投递行 | 失败 | 手动重试（视 `manual_retryable`） | 是 | `delivery_queries.rs:438-442` |

### [C2 附件上传与下载](messaging/C2-attachments.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | 未下载 | 远端 URL + 密钥 | 可下载 | 下载 | 否 | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | 开始下载（`transfer_state=1`） | 临时文件 | 下载中 | 等待 | 否 | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` 且本地文件存在 | 本地文件 | 可打开 | — | 是（文件丢失则回到 Remote 投影） | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | 下载失败，或启动 / 重启运行时发现中断 | `last_error_code` | 失败 | 重试 | 否 | `attachments.rs:462-469`, `:472-492` |
| C2-05 | 上传中 | 附件意图未完成 | — | `Uploading` | 文本 / 文件消息可撤销 | 否 | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | 暂停（作业内失败） | 上传失败写 `upload_state='paused'` | — | `Failed` | 可重试类错误允许重试；来源丢失、意图无效、不安全目标不可重试 | 否 | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 好友请求与 PQXDH 升级](messaging/C3-friend-requests.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（发出方） | 请求已提交入投递重试调度 | 待发请求 + 协议层待定会话 | 等待对方 | 撤回请求 | 否 | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 至少一处 relay 接受 | 同上 | 等待对方 | 撤回请求 | 否 | `A/types.rs:3512` |
| C3-03 | 被对方接受 | 收到对方回复 | 会话建立 | 联系人 | — | 是 | — |
| C3-04 | 撤回 | 用户撤回；或同一对端的新请求取代旧请求 | 清除协议层待定会话与传输 | — | — | 是 | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH 升级重试 | 用户手动触发 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` 分别提示 | 稍后重试 | 每次调用即返回 | `A/types.rs:533-540` |

### [C4 群邀请与 MLS Welcome 激活](messaging/C4-group-invites-mls.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | 发出或收到邀请；投递重试回到待定 | — | — | 接受 / 拒绝 / 撤回 | 否 | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | 投递失败、决定处理失败 | — | — | MLS 发出方可重试投递（`group_invite_lifecycle_allows_delivery_retry`） | 否 | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | 接受后等待成员激活控制消息落地 | — | `PendingAcknowledgement` | 等待 | 否 | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | 收到认证的激活证据或 relay OK | — | `Active` | — | 是 | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | 拒绝、撤回、过期 | — | — | — | 是 | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | 删除过程中待确认 | — | — | — | 否 | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · 身份、设备与恢复

### [D1 身份删除](identity-device-recovery/D1-identity-deletion.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | 用户确认删除，Core 写入删除计划；结束该身份的进行中通话 | 身份仍在，新工作被挡 | — | 回滚（只此阶段允许） | 否 | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | 平台确认已删钥匙串凭据 | 凭据已删，Core 数据待清 | — | 继续（可重复） | 否 | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | 旧版删除流程的中间点 | 同上 | — | 继续；确认旧秘密已删 | 否 | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | 删除完成 | Core 清理完成，删除行移除 | 身份消失 | — | — | 是 | — |

### [D2 设备激活、启动与备份恢复](identity-device-recovery/D2-device-activation-restore.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | 启动 | — | — | 等待 | 否 | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | 无数据 | — | — | 新建或恢复 | 否（待用户） | `:63` |
| D2-03 | `ActivationRequired{pending}` | 数据在，激活未验证 | — | — | 激活 | 否（待用户） | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | 恢复被挡、库密钥缺失、存储不可用 / 不兼容、钥匙串不可用、凭据冲突、策略不支持 | — | — | 按原因处理 | 否（待用户） | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | 一切就绪；或因手动 / 迁移源 / 迁移目标暂停 | — | — | — | 是（常驻） | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging`（iOS：`preparing` / `prepared`） | — | 暂存区；原数据未动 | — | 启动时清理暂存 | 否 | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain`（iOS：`replacement_started` / `restored_data_installed`） | — | 原数据快照 + 新数据 | — | 启动时回滚到快照，回滚后把日志记为 committed 以免重复回滚 | 否 | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking`（iOS：`validated`） | — | 新数据 | — | 同上（回滚） | 否 | same as above |
| D2-09 | `RestoreComplete`（iOS：`committed`） | — | 新数据，快照清理 | — | — | 是 | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed`（iOS：—） | — | 视失败点 | — | 只能回到 Idle | 是 | `A/backup_restore.rs:698-741` |

### [D3 钱包恢复](identity-device-recovery/D3-wallet-recovery.md)

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | 所有 epoch 完成 | 已恢复的 VTXO 与退出 | — | — | 是 | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | 服务器无此钱包记录 | — | — | 换源再试 | 是 | `types.rs:70` |
| D3-03 | `Incomplete` | 任一 epoch 不完整；账户保持 `recovery_pending`，合作操作被拒，预留为 None | 部分恢复 | — | 再次恢复 | 否 | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | 服务器不可用 / 服务器身份变了 | — | — | 稍后再试 / 人工判断 | 是（本次） | `types.rs:72-73` |
| D3-05 | 密钥扫描 `Ready` / `LimitReached` / `CandidatesMatched` | 按索引扫描到上限或命中 | 扫描进度 | — | 扩大扫描 | 否 | `types.rs:89-108` |
| D3-06 | BTC 历史扫描 `Ready` / `LimitReached` / `UnsupportedChainSource` | 同上（共享 BTC） | 已发现地址数 | — | 扩大扫描 | 否 | `types.rs:441-457` |
