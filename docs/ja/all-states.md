# 全状態（1 つの表）

[目次に戻る](README.md)

## A · オンチェーンと Ark の資金

### [A1 Ark への入金、BTC の直接送金、RBF による加速](ark-onchain/A1-onboard-direct-send-rbf.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | 見積もり生成と同時に永続化、有効期間 300 秒 | 未署名 PSBT。資金はオンチェーンウォレットにあり、ロックされていない | `wallet.bitcoin.review` | 確認する。または見積もりを破棄する（加速見積もりは削除可能） | いいえ（期限切れの自動削除なし。期限後の確認は拒否される） | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | 署名済み。`signed_tx_hex` を書き込み済み | 署名済み tx はローカルに保存。直接送金は BDK に `register_tx` 済みで使用済みとして計上。onboard の入力はロック済み | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | 待つのみ | いいえ | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | チェーンソースがメンプールにあると報告 | tx はメンプール内 | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | 待つ。直接送金は RBF 可能 | いいえ | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | onboard のみ：資金 tx は確定、Ark は未登録 | 資金は board 出力内。board VTXO は movement によりロック | 同上 + `wallet.bitcoin.waiting-ark` | 待つ | いいえ | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | チェーンソースの照会失敗、tx が見えない、onboard の共同署名なし、または movement が存在しない | 直接送金：署名済み tx は BDK に残る。onboard：入力はロック | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | 待つ | いいえ | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | 直接送金：いずれかの試行が確定。onboard：movement 成功かつ退出証明あり | 直接送金：相手に到達。onboard：Ark 内 | `wallet.bitcoin.completed` | — | はい（直接送金はチェーン再編成で Unknown に戻ることがある） | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | onboard のみ：サーバー条件の変更（`submit_signed_board` が None を返す）、または Bark movement が Failed/Canceled | 条件変更時：入力はロックされたまま。movement 失敗時：資金が退出フロー内にある可能性 | `wallet.bitcoin.failed` ("Transfer failed. Review a new quote to try again.") | 新しい見積もり | はい | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | 共同署名を取得、VTXO を構築済み | 資金 tx は送出待ち。VTXO は Locked として保存 | — | チェーン上で確認 → `Confirming`。確定 tx と入力が競合 → 失敗。それ以外は待機 | いいえ | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | 資金 tx はメンプールまたはブロック内 | board VTXO は Locked、`pending_board` に計上 | — | 確認数が足りたら登録。登録失敗時は再試行。期限が近づくと退出を開始 | いいえ | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed`（退出後） | VTXO が退出済み、または入力が競合 tx に使われた | 前者：退出フロー内。後者：資金は移動していない | — | 終了 | はい | `BK/actions/board.rs:299-314` |
| A1-11 | 加速見積もり `Reviewing` + `replacement` | ユーザーが加速を要求し、前提条件を満たす | 未署名の置換 PSBT、有効期間 300 秒 | — | 確認する。またはキャンセル（未署名レコードのみ削除） | いいえ（系譜に従う） | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | 親の `replaced_by` を設定、子を署名 | ユーザーが確認 | 親と子を同一トランザクションでコミット | — | 待つ | いいえ（系譜に従う） | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | 系譜の状態 | 確定した試行が Completed、他は変化なし。先頭はメンプール有無で判定 | — | — | — | いいえ（系譜に従う） | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 Ark からオンチェーンアドレスへの BTC 送金](ark-onchain/A2-ark-to-onchain-send.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing` (Bark `Start` / `SplitWithArkoor`) | ユーザーが確認。アクションのチェックポイントを書き込み | 入力 VTXO はアクションによりロック | `wallet.bitcoin-send.preparing` ("Payment in progress") | 「状態を確認」 | いいえ | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing` (Bark `ArkoorRegistrationRequired`) | arkoor 分割済み。元の入力は使用済みにマーク | 分割した送金 VTXO **とおつり** の両方がアクションによりロックされ、サーバー登録待ち | 同上 | 同上 | いいえ | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing` (Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`) | 登録済み。サーバーと offboard tx および forfeit を交換 | 送金 VTXO はロック。おつりはロック解除 | 同上 | 同上 | いいえ | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast` (Bark `AwaitingConfirmations`) | offboard tx を送出 | 資金はオンチェーン tx 内。VTXO は forfeit 済み | `wallet.bitcoin-send.broadcast` | 「状態を確認」 | いいえ | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | Movement 成功 | 受取人アドレスに到達 | `wallet.bitcoin-send.confirmed` | — | はい | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | Movement が Failed / Canceled（Bark は `Start`、`SplitWithArkoor`、forfeit 前の `ReadyForOffboard` の失敗時に入力を解放） | 入力は再び使用可能 | `wallet.bitcoin-send.failed` | 新しい見積もり | はい | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | 結果不明（確認呼び出しがタイムアウトまたはエラー） | `start_send_onchain_confirmed` はチェックポイントを書き込んだがタイムアウト | 基礎となる Bark の状態による | `wallet.bitcoin-send.uncertain` | 元の操作を確認する。新しい操作は開始できない | いいえ | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 出口手数料の予約](ark-onchain/A3-exit-fee-reservation.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`, `remaining_sats = Some(0)` | 新規アカウント登録。未請求の退出なし | 予約なし | なし | — | いいえ（再び Active になり得る） | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`, `Some(n)` | 退出が進行中で、手数料を見積もれる | 共有 BTC のうち n sat を予約扱いし、利用可能残高から差し引く | 残高構造に `reserved_sats` / `available_sats` がある（`W/ark/types.rs:429-439`） | 予約分に食い込む送金は明示的な確認が必要（`BitcoinExitReserveImpact`） | いいえ | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`、`Some(前回値)` | 見積もり失敗。最後の信頼できる値を保持 | 同上 | 同上 | 同上 | いいえ | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`, `None` | 復旧開始、新しい退出を復旧、Idle から初めて退出を発見、または再見積もりなしで新しい退出を承認 | 予約は不明。同じネットワークの直接送金と onboard は利用不可 | 送金がエラーになる | なし | いいえ | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | 登録待ちの復旧済み退出（`pending_recovered_exit_ids` が空でない） | 復旧した退出は記録済み、Bark のチェックポイントは未完了 | 同上 | 同上 | なし | いいえ | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 強制退出と請求](ark-onchain/A4-force-exit-and-claim.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | ユーザーが強制退出を確認、またはシステムが board / HTLC のために自動退出 | 退出チェーンの tx を段階的にブロードキャスト。共有 BTC が CPFP を支払う | `wallet.ark.exit-in-progress`, `wallet.ark.pending-funding`, `wallet.ark.refresh-to-progress-exit` | リフレッシュして進める。オンチェーン資金を追加 | いいえ | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | 全退出 tx が確定 | 退出出力。タイムロック未満了 | `wallet.ark.claim-timelock-running` | 待つ | いいえ | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | タイムロック満了 | 退出出力。ユーザーのみが使用可能 | `wallet.ark.claim-to-bitcoin-address` | 請求 | いいえ | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | 出力を使うメンプール tx が現れる | 請求 tx はメンプール内 | `wallet.ark.claim-in-progress` | 待つ | いいえ。メンプールから落ちると `Claimable` に戻る | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | 使用 tx が確定 | ユーザーのアドレスにある | — | — | はい | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO はオフチェーンで使用済み、またはサーバーが退出途中で回収 | なし | — | — | はい | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | 最終 tx のブロードキャスト前にキャンセル | VTXO はまだ使用可能 | — | 再度開始 | 再開可能 | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | ユーザーが見積もりを確認。署名済み tx を永続化。入力を `ark_claim_inputs` に登録 | — | `wallet.ark.claim-prepared` | 「元の tx を提出」 | いいえ | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | ブロードキャスト開始（状態を先に書き込む） | — | 請求中 | 待つ | いいえ | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | ブロードキャスト成功、またはチェーンソースがメンプールにあると報告 | — | `wallet.ark.claim-in-progress-message` | 「更新」 | いいえ | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired`（投影） | チェーン上で確定したが DB 書き込みに失敗 | — | `wallet.ark.claim-confirmation-checkpoint-required` | 更新 | いいえ | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | チェーン上で確定し、永続化済み | — | 請求済み | — | はい | `claim_store.rs:287-293` |

### [A5 リフレッシュと自動メンテナンス](ark-onchain/A5-refresh-and-maintenance.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | 予定リフレッシュがサーバーに受理され、目標高さを待つ。予定待機：はい（`is_scheduled_waiting`） | 入力はロックされていない。強制退出可能 | — | — | いいえ | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | 委任リフレッシュを提出し、サーバーのラウンドを待つ。予定待機：いいえ | 入力は movement によりロック、`pending_in_round` に計上 | — | — | いいえ | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | 対話型ラウンドの待機中 / 進行中。予定待機：いいえ | 同上 | — | — | いいえ | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished`（未完了） | 対話完了、ラウンドの資金 tx の確定待ち。予定待機：いいえ | 同上。forfeit は未完了 | — | — | いいえ | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished`（完了）→ `Confirmed` | 資金 tx 確定、forfeit 完了。予定待機：— | 新しい VTXO | — | — | はい | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | ラウンド失敗、ユーザーがキャンセル、または入力喪失で縮小不可。予定待機：— | 入力を解放 | — | — | はい | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | リフレッシュ不要 / 自動リフレッシュ無効 | — | `disabled-title` | 有効にする、または手動でリフレッシュ | いいえ（常駐） | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | 計画承認済み / 提出済み | — | — | 待つ | いいえ | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | 失敗時のバックオフ 60 秒 → 30 分 | — | — | 待つ | いいえ | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | 提出結果不明。先に復旧 | — | — | 待つ（`URGENT_CHECK_SECONDS` ごとに再確認） | いいえ | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | ユーザーによる手数料承認、手動確認、または資金追加が必要 | — | `authorization-*`、`funds-title` など | 承認、確認、または資金追加 | いいえ（ユーザー待ち） | `W/ark/maintenance.rs:35-62` |

## B · 支払いと受け取り

### [B1 Lightning の送受信（Ark）](payments/B1-lightning-ark.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | ユーザーが支払いを確認 | 入力 VTXO はアクションによりロック | `InProgress` | 待つ | いいえ | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | サーバーが HTLC 出力に共同署名 | HTLC VTXO はロック、`pending_lightning_send` に計上 | `InProgress` | 待つ | いいえ | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | サーバーに支払いを指示 | 同上 | `InProgress` | 待つ | いいえ | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | サーバーが失敗を報告、または HTLC 期限間近でも Pending | 同上。取り消しを準備 | `InProgress` | 待つ | いいえ | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | サーバーが取り消しを拒否 | 同上 | `InProgress{has_failed_revocation: true}` | 待つ | いいえ | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | プリイメージを取得 | 支払い済み | `Paid` | — | はい | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | 取り消し完了（Done） | サーバーが取り消しに共同署名 | 使用可能な VTXO に戻る | — | — | はい | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | インボイスを作成 | なし | — | — | いいえ | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | サーバーが HTLC-recv VTXO を準備 | ロック。まだ放棄可能 | — | — | いいえ。請求ウィンドウが閉じるか請求が拒否されると放棄 | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | プリイメージをサーバーに開示 | HTLC-recv VTXO はロック、`claimable_lightning_receive` に計上 | — | — | いいえ。請求失敗後は待機 | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | 請求出力を第三者に転送する必要がある | ロック | — | — | いいえ | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | 請求完了 | 新しい VTXO | — | — | はい | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Cashu の受け取り](payments/B2-cashu-receive.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | トークン受信、ミント登録済み | ベアラートークンはローカル暗号化ストアにある（唯一の再生可能なコピー） | 受け取り処理中 | 待つ | いいえ | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked` (`UnknownMint`) | ミント未登録 | 同上 | ミントの追加が必要 | ミントを追加して続行 | いいえ | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | 照合中、または SDK にこの操作の saga がある | トークン + CDK saga | 処理中 | 待つ | いいえ | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue` (`RetryRequired`) | 照合がエラーで SDK 操作 ID なし。30 秒後に再試行 | トークン | 処理中 | 待つ | いいえ | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal`（問題なし） | `Credited` / `AlreadyReceived` / `Recovered` を取得 | CDK 残高内 | 受け取り済み | — | はい | `receive_operations.rs:196-206` |
| B2-06 | `Terminal` (`AlreadySpent`) | ミントがトークン使用済みと応答 | なし | `CashuTokenAlreadySpent` エラー | — | はい | `receive_operations.rs:207-213` |

### [B3 チャット送金、紅包、受領通知](payments/B3-chat-transfer-red-packet.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`、プロバイダー操作なし | ユーザーが金額を確認 | 資金は未移動 | — | 続行 | いいえ | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`、プロバイダー操作あり（Cashu） | Cashu 送金 saga を開始 | ミント側の saga。トークンが既に存在する可能性 | `wallet.chat.previous-cashu-interrupted` | 「続行」：saga からトークンを回収 | いいえ | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`、Ark の結果未確定 | Ark 送金の結果が不明 | 既にアカウントから出ている可能性 | `wallet.chat.previous-wallet-transfer-interrupted` | tx ID で照合する。または送られていないことを確認して放棄 | いいえ | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | トークン（Cashu）または tx ID（Ark）を取得 | 資金はウォレットから出た。トークンはローカル saga 内 | `wallet.chat.wallet-transfer-complete`（メッセージの引き渡しを再開可能） | メッセージ送信を続ける | いいえ | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | 削除（完了） | メッセージを送信器に引き渡した | 引き渡しレコードを削除 | — | — | はい | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | なし → `Pending` / `Available` | 受信または送信 | — | 受け取り可能 | 受取人が受け取る。送信者は受け取れない（紅包） | いいえ | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | 受取人が受け取りをタップ。償還前に段階を記録 | — | 受け取り中、`can_retry` | 再試行 | いいえ | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | 償還して受領通知を送信。または相手の受領通知を受信 | — | 請求済み | 失敗した場合は受領通知を再送 | はい | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | 紅包の期限後に送信者が返金を開始 | — | `chat.payment.shared.refunding` / `continue-refund` | 返金を続ける（UI が自動再開） | いいえ | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | 返金完了 | — | 返金済み | 失敗した場合は受領通知を再送 | はい | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed`（送信者側の観測） | 送信者がトークン使用済みかつ受領通知なしを検出 | — | `payment_spent_without_receipt` | — | はい（投影） | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 リレー郵送料と回収](payments/B4-relay-postage-reclaim.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | リレーは郵送料を取らない | — | — | — | はい（その配信について） | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | 未払い、または支払えない | 未払い | — | — | いいえ | same as above |
| B4-03 | `Created` | 郵送料トークンを作成し、イベントとともにリレーへ送信 | トークンをリレーに渡し、ウォレット台帳に `relay_stamp` 送金を記録 | — | — | はい（その配信について） | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | 回収不可（結び付けなし） | 配信行の状態が failed / timeout / no_ack / terminal_failed / retry_exhausted のいずれでもない、または OK を受信済み | — | 入口を表示しない | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | 結び付けが `TokenCreated` の送金 saga 1 件にのみ対応 | トークンはまだリレーに換金されていない可能性 | 回収を表示 | 回収 | いいえ | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | 回収の意図を受理（ミント操作前に永続化）、またはミントの状態が不明 / 到達不能 | 意図 + 元の証明 | 処理中 | 再度回収 | いいえ | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | 交換完了 | ウォレットに戻った | 回収済み | — | はい | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | 全証明が使用済み（リレーが換金済み）、または一意に対応する台帳エントリなし | リレー側 | 回収不可 | — | はい | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 有料通話チケット](payments/B5-paid-call-tickets.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | ユーザーが購入を確認。注文行を書き込み | トークン未発行。または発行済みで未書き込み | 支払い中 | 続行 / キャンセル | いいえ | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | トークンを注文に書き込み notify に渡した | トークンは注文内 | 確認中。確認ページは Continue のみ | ポーリングを続ける | いいえ | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify が確認。その後利用状況で投影 | パス | 利用可能 / 使用中 / 使用済み | 通話 | はい（支払い面） | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify が失敗を返した、またはウォレットがトークン発行に失敗 | トークンを消去 | 失敗 | — | はい | `rtc_tickets.rs:118-125`, `:152` |

## C · メッセージと関係

### [C1 メッセージ送信](messaging/C1-message-send.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | リンクプレビューの意図が未完了 | 送信ジョブ | 送信中 | テキスト / ファイルは取り消し可能 | いいえ | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | 添付ファイルの意図が未完了 | 送信ジョブ + ローカル添付 | アップロード中 | 同上 | いいえ | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | ジョブあり、まだトランスポートに渡していない | 送信ジョブ | 送信中 | 同上 | いいえ | `delivery_queries.rs:430-432` |
| C1-04 | `Failed`（ジョブ内） | アップロード一時停止 | 送信ジョブ | 失敗 | 再試行 | いいえ | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | トランスポートイベントを作成。業務上の成功でも終了でもない | `app_message_transport_events` + リレーごとの配信行 | 送信中 | 待つ | いいえ | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | 主要ユニットが業務上の成功に到達 | 配信行 | 送信済み | — | はい | `delivery_queries.rs:433-437` |
| C1-07 | `Failed`（配信） | トランスポートポリシーが終了と判定 | 配信行 | 失敗 | 手動再試行（`manual_retryable` に従う） | はい | `delivery_queries.rs:438-442` |

### [C2 添付ファイルのアップロードとダウンロード](messaging/C2-attachments.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | 未ダウンロード | リモート URL + 鍵 | ダウンロード可能 | ダウンロード | いいえ | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | ダウンロード開始（`transfer_state=1`） | 一時ファイル | ダウンロード中 | 待つ | いいえ | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` かつローカルファイルが存在 | ローカルファイル | 開ける | — | はい（ファイルがなくなると Remote 投影に戻る） | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | ダウンロード失敗、または起動時 / ランタイム再起動時に中断を検出 | `last_error_code` | 失敗 | 再試行 | いいえ | `attachments.rs:462-469`, `:472-492` |
| C2-05 | アップロード中 | 添付ファイルの意図が未完了 | — | `Uploading` | テキスト / ファイルのメッセージは取り消し可能 | いいえ | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | 一時停止（ジョブ内の失敗） | アップロード失敗で `upload_state='paused'` を書き込み | — | `Failed` | 再試行可能なエラーは再試行可。ソース欠落、無効な意図、安全でない宛先は不可 | いいえ | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 友達リクエストと PQXDH アップグレード](messaging/C3-friend-requests.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（送信側） | リクエストを配信再試行スケジューラに投入 | 保留中のリクエスト + プロトコル層の保留セッション | 相手を待つ | 取り下げ | いいえ | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 少なくとも 1 つのリレーが受理 | 同上 | 相手を待つ | 取り下げ | いいえ | `A/types.rs:3512` |
| C3-03 | 相手が承認 | 相手の返信を受信 | セッション確立 | 連絡先 | — | はい | — |
| C3-04 | 取り下げ済み | ユーザーが取り下げ。または同じ相手への新しいリクエストが置き換え | プロトコルの保留セッションとトランスポートを消去 | — | — | はい | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH アップグレードの再試行 | ユーザーが手動で実行 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` ごとに別の表示 | 後で再試行 | 呼び出しごとに返る | `A/types.rs:533-540` |

### [C4 グループ招待と MLS Welcome の有効化](messaging/C4-group-invites-mls.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | 招待を送信または受信。配信再試行で pending に戻る | — | — | 承認 / 辞退 / 取り下げ | いいえ | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | 配信失敗、または決定の処理に失敗 | — | — | MLS の送信者は配信を再試行可能（`group_invite_lifecycle_allows_delivery_retry`） | いいえ | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | 承認済み。メンバー有効化の制御メッセージの到達待ち | — | `PendingAcknowledgement` | 待つ | いいえ | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | 認証済みの有効化証拠またはリレー OK を受信 | — | `Active` | — | はい | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | 辞退、取り下げ、期限切れ | — | — | — | はい | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | 削除中の確認待ち | — | — | — | いいえ | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · アイデンティティ、デバイス、復旧

### [D1 アイデンティティの削除](identity-device-recovery/D1-identity-deletion.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | ユーザーが削除を確認。Core が削除計画を書き込み、そのアイデンティティの進行中通話を終了 | アイデンティティはまだ存在。新しい処理はブロック | — | ロールバック（この段階のみ） | いいえ | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | プラットフォームがキーチェーン資格情報の削除を確認 | 資格情報は削除済み。Core データは清掃待ち | — | 続行（繰り返し可） | いいえ | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | 旧削除フローの中間点 | 同上 | — | 続行。旧シークレットの削除を確認 | いいえ | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | 削除完了 | Core の清掃完了。削除行を除去 | アイデンティティは消えた | — | — | はい | — |

### [D2 デバイスの有効化、起動、バックアップ復元](identity-device-recovery/D2-device-activation-restore.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | 起動 | — | — | 待つ | いいえ | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | データなし | — | — | 新規作成または復元 | いいえ（ユーザー待ち） | `:63` |
| D2-03 | `ActivationRequired{pending}` | データあり、有効化は未検証 | — | — | 有効化 | いいえ（ユーザー待ち） | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | 復元がブロック、DB 鍵がない、ストレージが利用不可 / 非互換、キーチェーン利用不可、資格情報の競合、未対応のポリシー | — | — | 理由に応じて対処 | いいえ（ユーザー待ち） | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | すべて準備完了。または手動 / 移行元 / 移行先として一時停止 | — | — | — | はい（常駐） | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging` (iOS: `preparing` / `prepared`) | — | ステージング領域。元のデータは変更なし | — | 起動時にステージングを清掃 | いいえ | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain` (iOS: `replacement_started` / `restored_data_installed`) | — | 元データのスナップショット + 新データ | — | 起動時にスナップショットへロールバックし、繰り返しを防ぐためジャーナルを committed にする | いいえ | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking` (iOS: `validated`) | — | 新データ | — | 同上（ロールバック） | いいえ | same as above |
| D2-09 | `RestoreComplete` (iOS: `committed`) | — | 新データ。スナップショットは清掃済み | — | — | はい | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed` (iOS: —) | — | 失敗箇所による | — | Idle に戻るのみ | はい | `A/backup_restore.rs:698-741` |

### [D3 ウォレットの復旧](identity-device-recovery/D3-wallet-recovery.md)

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | 全エポック完了 | 復旧した VTXO と退出 | — | — | はい | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | サーバーにこのウォレットの記録がない | — | — | 別のソースを試す | はい | `types.rs:70` |
| D3-03 | `Incomplete` | いずれかのエポックが未完了。アカウントは `recovery_pending` のまま、協調操作は拒否、予約は None | 一部復旧 | — | 再度復旧 | いいえ | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | サーバー利用不可 / サーバーの身元が変わった | — | — | 後で再試行 / 手動で判断 | はい（今回の試行） | `types.rs:72-73` |
| D3-05 | 鍵スキャン `Ready` / `LimitReached` / `CandidatesMatched` | 上限までスキャン、またはインデックスで一致 | スキャン進捗 | — | スキャンを拡大 | いいえ | `types.rs:89-108` |
| D3-06 | BTC 履歴スキャン `Ready` / `LimitReached` / `UnsupportedChainSource` | 同上（共有 BTC） | 発見したアドレス数 | — | スキャンを拡大 | いいえ | `types.rs:441-457` |
