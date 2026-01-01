# 全部狀態（一張表）

[返回目錄](README.md)

## A · 鏈上與 Ark 資金

### [A1 轉入 Ark、直接傳送 BTC、RBF 加速](ark-onchain/A1-onboard-direct-send-rbf.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | 報價生成即落庫，有效期 300 秒 | 未簽名 PSBT；資金仍在鏈上錢包，未加鎖 | `wallet.bitcoin.review` | 確認；或放棄報價（加速報價可刪除） | 否（無過期清理，過期後確認被拒） | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | 已簽名並寫入 `signed_tx_hex` | 簽名交易在本地；直接傳送已 `register_tx` 進 BDK，餘額按已花計；onboard 輸入被鎖定 | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | 只能等待 | 否 | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | 鏈源報 Mempool | 交易在記憶體池 | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | 等待；直接傳送可 RBF | 否 | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | 僅 onboard：資金交易已確認，Ark 尚未登記 | 資金在 board 輸出；board VTXO 被 movement 鎖定 | 同上 + `wallet.bitcoin.waiting-ark` | 等待 | 否 | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | 鏈源查詢失敗、看不到交易、onboard 未拿到聯籤或 movement 缺失 | 直接傳送：簽名交易仍在 BDK；onboard：輸入鎖定 | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | 等待 | 否 | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | 直接傳送：某次嘗試被確認；onboard：movement 成功且有退出憑據 | 直接傳送：已到對方；onboard：在 Ark | `wallet.bitcoin.completed` | — | 是（直接傳送遇鏈重組可回到 Unknown） | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | 僅 onboard：伺服器條款變化（`submit_signed_board` 返回 None），或 Bark movement 記為 Failed/Canceled | 條款變化時：輸入仍鎖定；movement 失敗時：資金可能處於退出流程中 | `wallet.bitcoin.failed`（"Transfer failed. Review a new quote to try again."） | 重新報價 | 是 | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | 拿到聯籤、VTXO 已構建 | 資金交易待推送；VTXO 以 Locked 存庫 | — | 見到上鍊 → `Confirming`；輸入被確認交易衝突 → 失敗；否則停泊等待 | 否 | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | 資金交易在記憶體池或區塊 | board VTXO Locked，計入 `pending_board` | — | 確認數夠即登記；登記失敗則重試；臨近到期自動發起退出 | 否 | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed`（退出後） | VTXO 已 Exited 或已被衝突花費 | 前者在退出流程中，後者資金未離開 | — | 終態 | 是 | `BK/actions/board.rs:299-314` |
| A1-11 | 加速報價 `Reviewing` + `replacement` | 使用者請求加速，條件滿足 | 未簽名替換 PSBT，300 秒有效 | — | 確認；或取消（只刪未簽名記錄） | 否（隨譜系） | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | 父記錄 `replaced_by` 置位、子記錄簽名 | 使用者確認 | 父子同一事務提交 | — | 等待 | 否（隨譜系） | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | 譜系狀態 | 任一嘗試確認即該嘗試 Completed，其餘維持；頭部按記憶體池/未見判定 | — | — | — | 否（隨譜系） | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 從 Ark 傳送 BTC 到鏈上地址](ark-onchain/A2-ark-to-onchain-send.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing`（Bark `Start` / `SplitWithArkoor`） | 使用者確認，動作檢查點寫入 | 輸入 VTXO 被動作鎖定 | `wallet.bitcoin-send.preparing`（"Payment in progress"） | 「檢查狀態」 | 否 | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing`（Bark `ArkoorRegistrationRequired`） | 已做 arkoor 拆分，原輸入標為已花 | 拆出的傳送 VTXO **與找零** 都被動作鎖定，等待向伺服器登記 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing`（Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`） | 登記成功，與伺服器交換 offboard 交易與 forfeit | 傳送 VTXO 鎖定；找零已解鎖 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast`（Bark `AwaitingConfirmations`） | offboard 交易已推送 | 資金在鏈上交易裡；VTXO 已 forfeit | `wallet.bitcoin-send.broadcast` | 「檢查狀態」 | 否 | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | movement Successful | 已到對方地址 | `wallet.bitcoin-send.confirmed` | — | 是 | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | movement Failed / Canceled（Bark 在 `Start`、`SplitWithArkoor`、未經 forfeit 的 `ReadyForOffboard` 失敗時釋放輸入） | 輸入回到可花 | `wallet.bitcoin-send.failed` | 重新報價 | 是 | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | 結果未知（確認呼叫超時或報錯） | `start_send_onchain_confirmed` 已寫檢查點但返回超時 | 同所在 Bark 狀態 | `wallet.bitcoin-send.uncertain` | 先查原操作，不能新建 | 否 | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 出口預留](ark-onchain/A3-exit-fee-reservation.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`，`remaining_sats = Some(0)` | 新賬戶登記；沒有未認領的退出 | 無預留 | 無 | — | 否（可再次進入 Active） | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`，`Some(n)` | 有進行中的退出，且能估算出手續費 | 共享 BTC 中 n sat 被視為預留，從可用餘額扣除 | 餘額結構裡有 `reserved_sats` / `available_sats`（`W/ark/types.rs:429-439`） | 使用者傳送時如侵佔預留須單獨確認（`BitcoinExitReserveImpact`） | 否 | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`，`Some(上次值)` | 估算失敗，保留上次可信值 | 同上 | 同上 | 同上 | 否 | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`，`None` | 恢復開始、恢復出新退出、從 Idle 首次發現退出、或授權新退出但不可重估 | 預留未知，同網路的直接傳送與 onboard 暫不可用 | 傳送報錯 | 無 | 否 | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | 待登記的恢復退出（`pending_recovered_exit_ids` 非空） | 恢復出的退出已登記、Bark 檢查點未齊 | 同上 | 同上 | 無 | 否 | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 強制退出與認領](ark-onchain/A4-force-exit-and-claim.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | 使用者確認強退，或系統為 board / HTLC 自動退出 | 退出鏈交易逐層廣播，需要共享 BTC 付 CPFP | `wallet.ark.exit-in-progress`、`wallet.ark.pending-funding`、`wallet.ark.refresh-to-progress-exit` | 重新整理推進；補充鏈上資金 | 否 | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | 全部退出交易確認 | 退出輸出，時間鎖未到 | `wallet.ark.claim-timelock-running` | 等待 | 否 | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | 時間鎖到期 | 退出輸出，僅使用者可花 | `wallet.ark.claim-to-bitcoin-address` | 認領 | 否 | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | 鏈上出現花費該輸出的記憶體池交易 | 認領交易在記憶體池 | `wallet.ark.claim-in-progress` | 等待 | 否；交易掉出記憶體池回到 `Claimable` | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | 花費交易確認 | 在使用者地址 | — | — | 是 | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO 已在鏈下被花；或伺服器在退出中途清掃 | 無 | — | — | 是 | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | 最終交易廣播前取消 | VTXO 仍可花 | — | 重新發起 | 可恢復 | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | 使用者確認報價，簽名交易落庫，輸入登記到 `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | 「提交原交易」 | 否 | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | 開始廣播（先改狀態再廣播） | — | 認領中 | 等待 | 否 | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | 廣播返回成功，或鏈源報記憶體池 | — | `wallet.ark.claim-in-progress-message` | 「重新整理」 | 否 | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired`（投影） | 鏈上已確認但寫庫失敗 | — | `wallet.ark.claim-confirmation-checkpoint-required` | 重新整理 | 否 | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | 鏈上確認並寫庫 | — | 已認領 | — | 是 | `claim_store.rs:287-293` |

### [A5 重新整理與自動維護](ark-onchain/A5-refresh-and-maintenance.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | 計劃重新整理已被伺服器接受，等待目標高度；計劃等待：是（`is_scheduled_waiting`） | 輸入未鎖，仍可強退 | — | — | 否 | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | 委託重新整理已提交，等待伺服器出輪次；計劃等待：否 | 輸入被 movement 鎖定，計入 `pending_in_round` | — | — | 否 | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | 互動式輪次排隊 / 進行中；計劃等待：否 | 同上 | — | — | 否 | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished`（未 done） | 互動結束，等待輪次資金交易確認；計劃等待：否 | 同上；forfeit 尚未完成 | — | — | 否 | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished`（done）→ `Confirmed` | 資金交易確認、forfeit 完成；計劃等待：— | 新 VTXO | — | — | 是 | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | 輪次失敗、使用者取消、輸入丟失無法縮減；計劃等待：— | 輸入釋放 | — | — | 是 | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | 無需重新整理 / 自動重新整理關閉 | — | `disabled-title` | 開啟或手動重新整理 | 否（常駐） | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | 計劃已批准 / 已提交 | — | — | 等待 | 否 | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | 失敗退避 60s→30min | — | — | 等待 | 否 | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | 提交結果不明，需先恢復 | — | — | 等待（每 `URGENT_CHECK_SECONDS` 再查） | 否 | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | 需使用者授權費用、人工複核、補資金 | — | `authorization-*`、`funds-title` 等 | 授權、複核或補資金 | 否（待使用者） | `W/ark/maintenance.rs:35-62` |

## B · 支付與收款

### [B1 Lightning 收發（Ark）](payments/B1-lightning-ark.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | 使用者確認支付 | 輸入 VTXO 被動作鎖定 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | 伺服器聯籤 HTLC 輸出 | HTLC VTXO 鎖定，計入 `pending_lightning_send` | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | 已通知伺服器付款 | 同上 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | 伺服器報失敗，或 HTLC 臨近到期仍 Pending | 同上，準備撤回 | `InProgress` | 等待 | 否 | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | 伺服器拒絕撤回 | 同上 | `InProgress{has_failed_revocation: true}` | 等待 | 否 | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | 拿到原像 | 已付出 | `Paid` | — | 是 | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | 撤回完成（Done） | 伺服器聯籤撤回 | 回到可花 VTXO | — | — | 是 | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | 生成發票 | 無 | — | — | 否 | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | 伺服器準備好 HTLC-recv VTXO | 鎖定，尚可放棄 | — | — | 否；認領視窗關閉或被拒即放棄 | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | 已向伺服器交出原像 | HTLC-recv VTXO 鎖定，計入 `claimable_lightning_receive` | — | — | 否；認領失敗後停泊 | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | 認領輸出需轉交第三方 | 鎖定 | — | — | 否 | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | 認領完成 | 新 VTXO | — | — | 是 | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Cashu 收款](payments/B2-cashu-receive.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | 收到 token，mint 已註冊 | bearer token 存於本地加密庫（唯一可重放副本） | 收款處理中 | 等待 | 否 | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked`（`UnknownMint`） | mint 未註冊 | 同上 | 需新增 mint | 新增 mint 後繼續 | 否 | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | 正在對賬，或 SDK 已有該操作的 saga | token + CDK saga | 處理中 | 等待 | 否 | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue`（`RetryRequired`） | 對賬出錯且沒有 SDK 操作 ID，30 秒後再試 | token | 處理中 | 等待 | 否 | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal`（issue 為空） | 收到 `Credited` / `AlreadyReceived` / `Recovered` | 已入 CDK 餘額 | 已到賬 | — | 是 | `receive_operations.rs:196-206` |
| B2-06 | `Terminal`（`AlreadySpent`） | mint 報 token 已被花 | 無 | `CashuTokenAlreadySpent` 報錯 | — | 是 | `receive_operations.rs:207-213` |

### [B3 聊天轉賬、紅包與收款回執](payments/B3-chat-transfer-red-packet.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`，無 provider 操作 | 使用者確認金額 | 未動資金 | — | 繼續 | 否 | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`，有 provider 操作（Cashu） | Cashu 傳送 saga 已開始 | mint 側 saga；token 可能已生成 | `wallet.chat.previous-cashu-interrupted` | 「繼續」：重新從 saga 找回 token | 否 | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`，Ark 未決 | Ark 轉賬結果未知 | 可能已離開賬戶 | `wallet.chat.previous-wallet-transfer-interrupted` | 輸入交易 ID 對賬；或確認未發出後放棄 | 否 | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | 拿到 token（Cashu）或交易 ID（Ark） | 資金已離開錢包，token 在本地 saga | `wallet.chat.wallet-transfer-complete`（可續交訊息） | 繼續把訊息發出 | 否 | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | 刪除（完成） | 訊息交給傳送器 | 交接記錄刪除 | — | — | 是 | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | 無 → `Pending` / `Available` | 收到或發出 | — | 可領取 | 收款方領取；傳送方（紅包）不可領 | 否 | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | 收款方點領取，先落階段再贖回 | — | 領取中，`can_retry` | 重試 | 否 | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | 贖回成功併發回執；或收到對方回執 | — | 已領取 | 回執失敗可重發 | 是 | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | 傳送方在紅包過期後開始退回 | — | `chat.payment.shared.refunding` / `continue-refund` | 繼續退回（介面自動續做） | 否 | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | 退回完成 | — | 已退回 | 回執失敗可重發 | 是 | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed`（傳送方觀察） | 傳送方發現 token 已被花但無回執 | — | `payment_spent_without_receipt` | — | 是（投影） | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 訊息郵資與找回](payments/B4-relay-postage-reclaim.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | relay 不收郵資 | — | — | — | 是（該投遞） | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | 尚未付或無法付 | 未付出 | — | — | 否 | same as above |
| B4-03 | `Created` | 已生成郵資 token 並隨事件發給 relay | token 交給 relay，錢包流水記 `relay_stamp` 傳送 | — | — | 是（該投遞） | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | 不可找回（無繫結） | 投遞行狀態不在 failed / timeout / no_ack / terminal_failed / retry_exhausted 之列，或曾收到 OK | — | 不顯示入口 | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | 繫結唯一對應一筆 `TokenCreated` 的傳送 saga | token 仍可能未被 relay 兌現 | 顯示找回 | 找回 | 否 | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | 已接受找回意圖（先落庫再動 mint），或 mint 狀態不明、連不上 | 意圖 + 原證明 | 處理中 | 再次找回 | 否 | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | 交換完成 | 回到錢包 | 已找回 | — | 是 | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | 證明全部已被花（relay 已兌現），或找不到唯一對應流水 | relay 處 | 不可找回 | — | 是 | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 付費通話票](payments/B5-paid-call-tickets.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | 使用者確認購買，訂單行寫入 | 尚未出 token；或已出 token 尚未寫入 | 付款中 | 繼續 / 取消 | 否 | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | token 寫入訂單並交給 notify | token 在訂單 | 確認中；確認頁只給 Continue | 繼續輪詢 | 否 | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify 確認；隨後按使用情況投影 | 通行證 | 可用 / 使用中 / 已用 | 撥打 | 是（支付面） | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify 回失敗，或錢包出 token 失敗 | token 被置空 | 失敗 | — | 是 | `rtc_tickets.rs:118-125`, `:152` |

## C · 訊息與關係

### [C1 訊息傳送](messaging/C1-message-send.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | 有連結預覽意圖未完成 | 傳送作業 | 傳送中 | 文字 / 檔案可撤銷 | 否 | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | 附件意圖未完成 | 傳送作業 + 本地附件 | 上傳中 | 同上 | 否 | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | 作業存在，尚未交給傳輸層 | 傳送作業 | 傳送中 | 同上 | 否 | `delivery_queries.rs:430-432` |
| C1-04 | `Failed`（作業內） | 上傳被暫停 | 傳送作業 | 失敗 | 重試 | 否 | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | 已生成傳輸事件，未達業務成功也未終態 | `app_message_transport_events` + 每 relay 投遞行 | 傳送中 | 等待 | 否 | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | 主要單元達到業務成功 | 投遞行 | 已傳送 | — | 是 | `delivery_queries.rs:433-437` |
| C1-07 | `Failed`（投遞） | 傳輸策略判終態 | 投遞行 | 失敗 | 手動重試（視 `manual_retryable`） | 是 | `delivery_queries.rs:438-442` |

### [C2 附件上傳與下載](messaging/C2-attachments.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | 未下載 | 遠端 URL + 金鑰 | 可下載 | 下載 | 否 | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | 開始下載（`transfer_state=1`） | 臨時檔案 | 下載中 | 等待 | 否 | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` 且本地檔案存在 | 本地檔案 | 可開啟 | — | 是（檔案丟失則回到 Remote 投影） | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | 下載失敗，或啟動 / 重啟執行時發現中斷 | `last_error_code` | 失敗 | 重試 | 否 | `attachments.rs:462-469`, `:472-492` |
| C2-05 | 上傳中 | 附件意圖未完成 | — | `Uploading` | 文字 / 檔案訊息可撤銷 | 否 | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | 暫停（作業內失敗） | 上傳失敗寫 `upload_state='paused'` | — | `Failed` | 可重試類錯誤允許重試；來源丟失、意圖無效、不安全目標不可重試 | 否 | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 好友請求與 PQXDH 升級](messaging/C3-friend-requests.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（發出方） | 請求已提交入投遞重試排程 | 待發請求 + 協議層待定會話 | 等待對方 | 撤回請求 | 否 | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 至少一處 relay 接受 | 同上 | 等待對方 | 撤回請求 | 否 | `A/types.rs:3512` |
| C3-03 | 被對方接受 | 收到對方回覆 | 會話建立 | 聯絡人 | — | 是 | — |
| C3-04 | 撤回 | 使用者撤回；或同一對端的新請求取代舊請求 | 清除協議層待定會話與傳輸 | — | — | 是 | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH 升級重試 | 使用者手動觸發 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` 分別提示 | 稍後重試 | 每次呼叫即返回 | `A/types.rs:533-540` |

### [C4 群邀請與 MLS Welcome 啟用](messaging/C4-group-invites-mls.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | 發出或收到邀請；投遞重試回到待定 | — | — | 接受 / 拒絕 / 撤回 | 否 | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | 投遞失敗、決定處理失敗 | — | — | MLS 發出方可重試投遞（`group_invite_lifecycle_allows_delivery_retry`） | 否 | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | 接受後等待成員啟用控制訊息落地 | — | `PendingAcknowledgement` | 等待 | 否 | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | 收到認證的啟用證據或 relay OK | — | `Active` | — | 是 | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | 拒絕、撤回、過期 | — | — | — | 是 | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | 刪除過程中待確認 | — | — | — | 否 | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · 身份、裝置與恢復

### [D1 身份刪除](identity-device-recovery/D1-identity-deletion.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | 使用者確認刪除，Core 寫入刪除計劃；結束該身份的進行中通話 | 身份仍在，新工作被擋 | — | 回滾（只此階段允許） | 否 | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | 平臺確認已刪鑰匙串憑據 | 憑據已刪，Core 資料待清 | — | 繼續（可重複） | 否 | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | 舊版刪除流程的中間點 | 同上 | — | 繼續；確認舊秘密已刪 | 否 | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | 刪除完成 | Core 清理完成，刪除行移除 | 身份消失 | — | — | 是 | — |

### [D2 裝置啟用、啟動與備份恢復](identity-device-recovery/D2-device-activation-restore.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | 啟動 | — | — | 等待 | 否 | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | 無資料 | — | — | 新建或恢復 | 否（待使用者） | `:63` |
| D2-03 | `ActivationRequired{pending}` | 資料在，啟用未驗證 | — | — | 啟用 | 否（待使用者） | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | 恢復被擋、庫金鑰缺失、儲存不可用 / 不相容、鑰匙串不可用、憑據衝突、策略不支援 | — | — | 按原因處理 | 否（待使用者） | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | 一切就緒；或因手動 / 遷移源 / 遷移目標暫停 | — | — | — | 是（常駐） | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging`（iOS：`preparing` / `prepared`） | — | 暫存區；原資料未動 | — | 啟動時清理暫存 | 否 | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain`（iOS：`replacement_started` / `restored_data_installed`） | — | 原資料快照 + 新資料 | — | 啟動時回滾到快照，回滾後把日誌記為 committed 以免重複回滾 | 否 | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking`（iOS：`validated`） | — | 新資料 | — | 同上（回滾） | 否 | same as above |
| D2-09 | `RestoreComplete`（iOS：`committed`） | — | 新資料，快照清理 | — | — | 是 | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed`（iOS：—） | — | 視失敗點 | — | 只能回到 Idle | 是 | `A/backup_restore.rs:698-741` |

### [D3 錢包恢復](identity-device-recovery/D3-wallet-recovery.md)

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | 所有 epoch 完成 | 已恢復的 VTXO 與退出 | — | — | 是 | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | 伺服器無此錢包記錄 | — | — | 換源再試 | 是 | `types.rs:70` |
| D3-03 | `Incomplete` | 任一 epoch 不完整；賬戶保持 `recovery_pending`，合作操作被拒，預留為 None | 部分恢復 | — | 再次恢復 | 否 | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | 伺服器不可用 / 伺服器身份變了 | — | — | 稍後再試 / 人工判斷 | 是（本次） | `types.rs:72-73` |
| D3-05 | 金鑰掃描 `Ready` / `LimitReached` / `CandidatesMatched` | 按索引掃描到上限或命中 | 掃描進度 | — | 擴大掃描 | 否 | `types.rs:89-108` |
| D3-06 | BTC 歷史掃描 `Ready` / `LimitReached` / `UnsupportedChainSource` | 同上（共享 BTC） | 已發現地址數 | — | 擴大掃描 | 否 | `types.rs:441-457` |
