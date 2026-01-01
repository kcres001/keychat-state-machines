# All states (single table)

[Back to contents](README.md)

## A · On-chain & Ark funds

### [A1 Onboard to Ark, direct BTC send, RBF acceleration](ark-onchain/A1-onboard-direct-send-rbf.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | Persisted as soon as the quote is generated; valid 300 s | Unsigned PSBT; funds still in the on-chain wallet, not locked | `wallet.bitcoin.review` | Confirm; or drop the quote (acceleration quotes can be deleted) | No (no expiry cleanup; confirming after expiry is rejected) | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | Signed; `signed_tx_hex` written | Signed tx stored locally; direct send already `register_tx`-ed into BDK and counted as spent; onboard inputs locked | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | Wait only | No | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | Chain source reports mempool | Tx in mempool | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | Wait; direct send can RBF | No | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | Onboard only: funding tx confirmed, Ark has not registered it | Funds in board output; board VTXO locked by the movement | Same as above + `wallet.bitcoin.waiting-ark` | Wait | No | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | Chain-source query failed, tx not visible, onboard has no cosignature, or movement missing | Direct send: signed tx still in BDK; onboard: inputs locked | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | Wait | No | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | Direct send: one attempt confirmed; onboard: movement successful with exit proof | Direct send: delivered; onboard: in Ark | `wallet.bitcoin.completed` | — | Yes (direct send can return to Unknown on a chain reorg) | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | Onboard only: server terms changed (`submit_signed_board` returns None), or Bark movement Failed/Canceled | On terms change: inputs remain locked; on movement failure: funds may be inside an exit | `wallet.bitcoin.failed` ("Transfer failed. Review a new quote to try again.") | New quote | Yes | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | Cosignature obtained, VTXO built | Funding tx to be pushed; VTXO stored as Locked | — | Seen on chain → `Confirming`; inputs conflicted by a confirmed tx → fail; otherwise park | No | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | Funding tx in mempool or block | Board VTXO Locked, counted in `pending_board` | — | Register once confirmations suffice; retry on registration failure; start an exit near expiry | No | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed` (after exit) | VTXO exited, or inputs spent by a conflicting tx | Former: in an exit; latter: funds never left | — | Terminal | Yes | `BK/actions/board.rs:299-314` |
| A1-11 | Acceleration quote `Reviewing` + `replacement` | User requests acceleration; preconditions met | Unsigned replacement PSBT, valid 300 s | — | Confirm; or cancel (deletes only the unsigned record) | No (follows lineage) | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | Parent `replaced_by` set, child signed | User confirms | Parent and child committed in one transaction | — | Wait | No (follows lineage) | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | Lineage state | The attempt that confirms becomes Completed, others unchanged; head judged by mempool / not-seen | — | — | — | No (follows lineage) | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 Sending BTC from Ark to an on-chain address](ark-onchain/A2-ark-to-onchain-send.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing` (Bark `Start` / `SplitWithArkoor`) | User confirms; action checkpoint written | Input VTXOs locked by the action | `wallet.bitcoin-send.preparing` ("Payment in progress") | "Check status" | No | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing` (Bark `ArkoorRegistrationRequired`) | Arkoor split done; original inputs marked spent | Split-off send VTXO **and change** both locked by the action, awaiting server registration | Same as above | Same as above | No | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing` (Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`) | Registered; offboard tx and forfeits exchanged with server | Send VTXO locked; change unlocked | Same as above | Same as above | No | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast` (Bark `AwaitingConfirmations`) | Offboard tx pushed | Funds in an on-chain tx; VTXOs forfeited | `wallet.bitcoin-send.broadcast` | "Check status" | No | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | Movement Successful | Delivered to recipient address | `wallet.bitcoin-send.confirmed` | — | Yes | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | Movement Failed / Canceled (Bark releases inputs on failure in `Start`, `SplitWithArkoor`, or a pre-forfeit `ReadyForOffboard`) | Inputs spendable again | `wallet.bitcoin-send.failed` | New quote | Yes | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | Outcome unknown (confirm call timed out or errored) | `start_send_onchain_confirmed` wrote the checkpoint but timed out | Whatever the underlying Bark state holds | `wallet.bitcoin-send.uncertain` | Check the original operation; cannot start another | No | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 Exit fee reservation](ark-onchain/A3-exit-fee-reservation.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`, `remaining_sats = Some(0)` | New account registered; no unclaimed exits | No reservation | None | — | No (can become Active again) | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`, `Some(n)` | Exit in progress and fees can be estimated | n sat of shared BTC treated as reserved, deducted from available balance | Balance struct has `reserved_sats` / `available_sats` (`W/ark/types.rs:429-439`) | Sends that eat into the reserve need explicit confirmation (`BitcoinExitReserveImpact`) | No | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`, `Some(last value)` | Estimation failed; last trusted value kept | Same as above | Same as above | Same as above | No | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`, `None` | Recovery started, recovered a new exit, first exit discovered from Idle, or new exit authorized without re-estimation | Reservation unknown; direct sends and onboards on the same network are unavailable | Send errors | None | No | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | Pending recovered exits (`pending_recovered_exit_ids` non-empty) | Recovered exits recorded, Bark checkpoints incomplete | Same as above | Same as above | None | No | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 Force exit and claim](ark-onchain/A4-force-exit-and-claim.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | User confirms force exit, or system auto-exits for a board / HTLC | Exit-chain txs broadcast level by level; shared BTC pays CPFP | `wallet.ark.exit-in-progress`, `wallet.ark.pending-funding`, `wallet.ark.refresh-to-progress-exit` | Refresh to progress; add on-chain funds | No | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | All exit txs confirmed | Exit output, timelock not yet expired | `wallet.ark.claim-timelock-running` | Wait | No | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | Timelock expired | Exit output, spendable only by the user | `wallet.ark.claim-to-bitcoin-address` | Claim | No | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | A mempool tx spending the output appears | Claim tx in mempool | `wallet.ark.claim-in-progress` | Wait | No; falls back to `Claimable` if it drops from the mempool | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | Spending tx confirmed | At the user address | — | — | Yes | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO already spent off-chain; or server swept mid-exit | None | — | — | Yes | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | Canceled before the final tx is broadcast | VTXO still spendable | — | Start again | Resumable | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | User confirms quote; signed tx persisted; inputs registered in `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | "Submit original tx" | No | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | Broadcast started (state written before broadcasting) | — | Claiming | Wait | No | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | Broadcast returned success, or chain source reports mempool | — | `wallet.ark.claim-in-progress-message` | "Refresh" | No | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired` (projection) | Confirmed on chain but DB write failed | — | `wallet.ark.claim-confirmation-checkpoint-required` | Refresh | No | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | Confirmed on chain and persisted | — | Claimed | — | Yes | `claim_store.rs:287-293` |

### [A5 Refresh and automatic maintenance](ark-onchain/A5-refresh-and-maintenance.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | Scheduled refresh accepted by server, waiting for target height; scheduled-waiting: yes (`is_scheduled_waiting`) | Inputs not locked; still force-exitable | — | — | No | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | Delegated refresh submitted, waiting for the server to run a round; scheduled-waiting: no | Inputs locked by the movement, counted in `pending_in_round` | — | — | No | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | Interactive round queued / in progress; scheduled-waiting: no | Same as above | — | — | No | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished` (not done) | Interaction done, waiting for round funding tx to confirm; scheduled-waiting: no | Same as above; forfeits not yet complete | — | — | No | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished` (done) → `Confirmed` | Funding tx confirmed and forfeits complete; scheduled-waiting: — | New VTXOs | — | — | Yes | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | Round failed, user canceled, or inputs lost and cannot shrink; scheduled-waiting: — | Inputs released | — | — | Yes | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | No refresh needed / auto refresh off | — | `disabled-title` | Enable, or refresh manually | No (resting) | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | Plan approved / submitted | — | — | Wait | No | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | Failure backoff 60 s → 30 min | — | — | Wait | No | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | Submission outcome unknown; recovery first | — | — | Wait (re-checked every `URGENT_CHECK_SECONDS`) | No | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | Needs user fee authorization, manual review, or more funds | — | `authorization-*`, `funds-title`, etc. | Authorize, review, or add funds | No (awaiting user) | `W/ark/maintenance.rs:35-62` |

## B · Payments

### [B1 Lightning send and receive (Ark)](payments/B1-lightning-ark.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | User confirms payment | Input VTXOs locked by the action | `InProgress` | Wait | No | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | Server cosigned HTLC outputs | HTLC VTXOs locked, counted in `pending_lightning_send` | `InProgress` | Wait | No | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | Server told to pay | Same as above | `InProgress` | Wait | No | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | Server reports failure, or still Pending near HTLC expiry | Same as above, revocation prepared | `InProgress` | Wait | No | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | Server refuses to revoke | Same as above | `InProgress{has_failed_revocation: true}` | Wait | No | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | Preimage obtained | Paid out | `Paid` | — | Yes | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | Revocation done (Done) | Server cosigns the revocation | Back to spendable VTXOs | — | — | Yes | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | Invoice created | None | — | — | No | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | Server prepared HTLC-recv VTXOs | Locked; still abandonable | — | — | No; abandoned if claim window closes or claim rejected | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | Preimage revealed to server | HTLC-recv VTXOs locked, counted in `claimable_lightning_receive` | — | — | No; parks after a failed claim | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | Claim outputs must be forwarded to a third party | Locked | — | — | No | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | Claim complete | New VTXOs | — | — | Yes | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Cashu receive](payments/B2-cashu-receive.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | Token received, mint registered | Bearer token in local encrypted store (the only replayable copy) | Receive in progress | Wait | No | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked` (`UnknownMint`) | Mint not registered | Same as above | Add mint required | Add the mint, then continue | No | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | Reconciling, or SDK already has a saga for it | Token + CDK saga | Processing | Wait | No | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue` (`RetryRequired`) | Reconcile errored with no SDK operation id; retry after 30 s | Token | Processing | Wait | No | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal` (no issue) | Got `Credited` / `AlreadyReceived` / `Recovered` | In CDK balance | Received | — | Yes | `receive_operations.rs:196-206` |
| B2-06 | `Terminal` (`AlreadySpent`) | Mint says token already spent | None | `CashuTokenAlreadySpent` error | — | Yes | `receive_operations.rs:207-213` |

### [B3 Chat transfers, red packets and receipts](payments/B3-chat-transfer-red-packet.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`, no provider operation | User confirms amount | No funds moved | — | Continue | No | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`, with provider operation (Cashu) | Cashu send saga started | Mint-side saga; token may already exist | `wallet.chat.previous-cashu-interrupted` | "Continue": recover the token from the saga | No | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`, Ark outcome unresolved | Ark transfer outcome unknown | May already have left the account | `wallet.chat.previous-wallet-transfer-interrupted` | Reconcile by tx id; or abandon after confirming nothing was sent | No | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | Token (Cashu) or tx id (Ark) obtained | Funds have left the wallet; token in local saga | `wallet.chat.wallet-transfer-complete` (message hand-off can resume) | Continue sending the message | No | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | Deleted (done) | Message handed to the sender | Hand-off record deleted | — | — | Yes | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | none → `Pending` / `Available` | Received or sent | — | Claimable | Recipient claims; sender cannot claim (red packet) | No | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | Recipient taps claim; phase persisted before redeeming | — | Claiming, `can_retry` | Retry | No | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | Redeemed and receipt sent; or peer receipt received | — | Claimed | Resend receipt if it failed | Yes | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | Sender starts refund after red packet expiry | — | `chat.payment.shared.refunding` / `continue-refund` | Continue refund (UI auto-resumes) | No | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | Refund complete | — | Refunded | Resend receipt if it failed | Yes | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed` (sender-observed) | Sender finds token spent with no receipt | — | `payment_spent_without_receipt` | — | Yes (projection) | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 Relay postage and reclaim](payments/B4-relay-postage-reclaim.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | Relay charges no postage | — | — | — | Yes (for that delivery) | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | Not yet paid, or cannot pay | Not paid | — | — | No | same as above |
| B4-03 | `Created` | Postage token created and sent to relay with the event | Token handed to relay; wallet ledger records a `relay_stamp` send | — | — | Yes (for that delivery) | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | Not reclaimable (no binding) | Delivery row status not in failed / timeout / no_ack / terminal_failed / retry_exhausted, or an OK was received | — | No entry shown | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | Binding maps to exactly one `TokenCreated` send saga | Token possibly not yet redeemed by relay | Reclaim shown | Reclaim | No | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | Reclaim intent accepted (persisted before touching the mint), or mint state unclear / unreachable | Intent + original proofs | Processing | Reclaim again | No | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | Swap complete | Back in wallet | Reclaimed | — | Yes | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | All proofs spent (relay redeemed), or no unique matching ledger entry | At the relay | Not reclaimable | — | Yes | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 Paid call tickets](payments/B5-paid-call-tickets.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | User confirms purchase; order row written | Token not yet issued; or issued but not yet written | Paying | Continue / cancel | No | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | Token written to the order and given to notify | Token in the order | Confirming; confirm page offers only Continue | Keep polling | No | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify confirmed; then projected by usage | Pass | Available / in use / used | Call | Yes (payment side) | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify returned failure, or wallet failed to issue a token | Token cleared | Failed | — | Yes | `rtc_tickets.rs:118-125`, `:152` |

## C · Messaging & relationships

### [C1 Message send](messaging/C1-message-send.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | Link-preview intent unfinished | Send job | Sending | Text / file can be canceled | No | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | Attachment intent unfinished | Send job + local attachments | Uploading | Same as above | No | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | Job exists, not yet handed to transport | Send job | Sending | Same as above | No | `delivery_queries.rs:430-432` |
| C1-04 | `Failed` (inside job) | Upload paused | Send job | Failed | Retry | No | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | Transport events created; neither business success nor terminal | `app_message_transport_events` + per-relay delivery rows | Sending | Wait | No | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | Primary unit reached business success | Delivery rows | Sent | — | Yes | `delivery_queries.rs:433-437` |
| C1-07 | `Failed` (delivery) | Transport policy judged terminal | Delivery rows | Failed | Manual retry (per `manual_retryable`) | Yes | `delivery_queries.rs:438-442` |

### [C2 Attachment upload and download](messaging/C2-attachments.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | Not downloaded | Remote URL + key | Downloadable | Download | No | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | Download started (`transfer_state=1`) | Temp file | Downloading | Wait | No | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` and local file exists | Local file | Openable | — | Yes (back to Remote projection if the file is gone) | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | Download failed, or interruption found at startup / runtime restart | `last_error_code` | Failed | Retry | No | `attachments.rs:462-469`, `:472-492` |
| C2-05 | Uploading | Attachment intent unfinished | — | `Uploading` | Text / file messages can be canceled | No | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | Paused (failure inside job) | Upload failure writes `upload_state='paused'` | — | `Failed` | Retry allowed for retryable errors; not for missing source, invalid intent, unsafe target | No | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 Friend requests and PQXDH upgrade](messaging/C3-friend-requests.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending` (sender side) | Request submitted to the delivery retry scheduler | Pending request + protocol-level pending session | Waiting for peer | Withdraw | No | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | At least one relay accepted | Same as above | Waiting for peer | Withdraw | No | `A/types.rs:3512` |
| C3-03 | Accepted by peer | Peer reply received | Session established | Contact | — | Yes | — |
| C3-04 | Withdrawn | User withdraws; or a new request to the same peer supersedes it | Protocol pending session and transport cleared | — | — | Yes | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH upgrade retry | User triggers manually | — | Separate prompts for `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` | Retry later | Returns per call | `A/types.rs:533-540` |

### [C4 Group invites and MLS Welcome activation](messaging/C4-group-invites-mls.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | Invite sent or received; delivery retry returns to pending | — | — | Accept / decline / withdraw | No | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | Delivery failed, or decision processing failed | — | — | MLS sender can retry delivery (`group_invite_lifecycle_allows_delivery_retry`) | No | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | Accepted; waiting for member-activation control message to land | — | `PendingAcknowledgement` | Wait | No | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | Authenticated activation evidence or relay OK received | — | `Active` | — | Yes | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | Declined, withdrawn, expired | — | — | — | Yes | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | Pending confirmation during deletion | — | — | — | No | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · Identity, device & recovery

### [D1 Identity deletion](identity-device-recovery/D1-identity-deletion.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | User confirms deletion; Core writes deletion plan; active calls of the identity ended | Identity still present; new work blocked | — | Roll back (only in this phase) | No | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | Platform confirms keychain credentials deleted | Credentials deleted; Core data pending cleanup | — | Continue (repeatable) | No | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Intermediate points of the legacy deletion flow | Same as above | — | Continue; confirm legacy secret deleted | No | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Deletion complete | Core cleanup done; deletion row removed | Identity gone | — | — | Yes | — |

### [D2 Device activation, startup and backup restore](identity-device-recovery/D2-device-activation-restore.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | Launch | — | — | Wait | No | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | No data | — | — | Create or restore | No (awaiting user) | `:63` |
| D2-03 | `ActivationRequired{pending}` | Data present, activation not verified | — | — | Activate | No (awaiting user) | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | Restore blocked, DB key missing, storage unavailable / incompatible, keychain unavailable, conflicting credentials, unsupported policy | — | — | Act per reason | No (awaiting user) | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | All ready; or paused manually / as migration source / as migration target | — | — | — | Yes (resting) | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging` (iOS: `preparing` / `prepared`) | — | Staging area; original data untouched | — | Staging cleaned up at startup | No | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain` (iOS: `replacement_started` / `restored_data_installed`) | — | Original-data snapshot + new data | — | Rolled back to snapshot at startup; journal then marked committed to avoid repeat rollback | No | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking` (iOS: `validated`) | — | New data | — | Same as above (rollback) | No | same as above |
| D2-09 | `RestoreComplete` (iOS: `committed`) | — | New data; snapshot cleaned | — | — | Yes | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed` (iOS: —) | — | Depends on failure point | — | Can only return to Idle | Yes | `A/backup_restore.rs:698-741` |

### [D3 Wallet recovery](identity-device-recovery/D3-wallet-recovery.md)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | All epochs complete | Recovered VTXOs and exits | — | — | Yes | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | Server has no record of the wallet | — | — | Try another source | Yes | `types.rs:70` |
| D3-03 | `Incomplete` | Any epoch incomplete; account stays `recovery_pending`, cooperative operations refused, reservation None | Partially recovered | — | Recover again | No | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | Server unavailable / server identity changed | — | — | Retry later / judge manually | Yes (this attempt) | `types.rs:72-73` |
| D3-05 | Key scan `Ready` / `LimitReached` / `CandidatesMatched` | Scanned to limit or matched by index | Scan progress | — | Extend scan | No | `types.rs:89-108` |
| D3-06 | BTC history scan `Ready` / `LimitReached` / `UnsupportedChainSource` | Same as above (shared BTC) | Addresses discovered | — | Extend scan | No | `types.rs:441-457` |
