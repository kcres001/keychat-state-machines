# A2 從 Ark 傳送 BTC 到鏈上地址

[返回目錄](../README.md) · A · 鏈上與 Ark 資金

## 狀態圖

GitHub 會直接渲染下方的 Mermaid 程式碼塊；如果顯示異常，請開啟 SVG 圖。

```mermaid
flowchart TD
  Q(("開始")) -->|"【使用者】確認報價"| ST["Start<br/>顯示 Preparing"]
  ST -->|"【本地】鎖定輸入"| SP["SplitWithArkoor"]
  ST -->|"【本地】鎖定失敗"| F1["Failed"]
  SP -->|"【伺服器】arkoor 聯籤，原輸入標為已花"| ARR["ArkoorRegistrationRequired<br/>傳送 VTXO 與找零由動作鎖定"]
  SP -->|"【伺服器】拒絕"| F2["Failed<br/>輸入釋放"]
  ARR -->|"【伺服器】登記成功，找零解鎖"| RFO["ReadyForOffboard"]
  ARR -.- ARRL1(["⟲ 【伺服器】登記失敗 → 重試"])
  class ARRL1 loop
  RFO -->|"【伺服器】構建 offboard 交易"| OTP["OffboardTxPrepared"]
  RFO -->|"【伺服器】拒絕"| F2
  OTP -->|"【伺服器】交換 forfeit"| RFB["ReadyForBroadcast"]
  OTP -->|"【伺服器】會話丟失，交易已上鍊"| AC
  OTP -->|"【伺服器】會話丟失，交易未上鍊"| RFOP["ReadyForOffboard<br/>帶 prior_txid"]
  RFOP -->|"【伺服器】因費率等拒絕"| F2
  RFOP -.- RFOPL2(["⟲ 【伺服器】其他拒絕 → 停泊，繼續查詢舊交易"])
  class RFOPL2 loop
  RFOP -->|"【鏈】舊交易出現"| AC
  RFB -->|"【本地】廣播"| AC["AwaitingConfirmations<br/>顯示 Broadcast"]
  RFB -.- RFBL3(["⟲ 【伺服器】拒絕 → 停泊"])
  class RFBL3 loop
  AC -->|"【鏈】確認"| OK["Confirmed"]
  AC -.- ACL4(["⟲ 【鏈源】長期看不到 → 停泊，輸入保持鎖定"])
  class ACL4 loop
  class OK,F1,F2 term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 圖](diagrams/A2.svg) · [Mermaid 源](diagrams/A2.mmd)

## 狀態表

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing`（Bark `Start` / `SplitWithArkoor`） | 使用者確認，動作檢查點寫入 | 輸入 VTXO 被動作鎖定 | `wallet.bitcoin-send.preparing`（"Payment in progress"） | 「檢查狀態」 | 否 | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing`（Bark `ArkoorRegistrationRequired`） | 已做 arkoor 拆分，原輸入標為已花 | 拆出的傳送 VTXO **與找零** 都被動作鎖定，等待向伺服器登記 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing`（Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`） | 登記成功，與伺服器交換 offboard 交易與 forfeit | 傳送 VTXO 鎖定；找零已解鎖 | 同上 | 同上 | 否 | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast`（Bark `AwaitingConfirmations`） | offboard 交易已推送 | 資金在鏈上交易裡；VTXO 已 forfeit | `wallet.bitcoin-send.broadcast` | 「檢查狀態」 | 否 | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | movement Successful | 已到對方地址 | `wallet.bitcoin-send.confirmed` | — | 是 | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | movement Failed / Canceled（Bark 在 `Start`、`SplitWithArkoor`、未經 forfeit 的 `ReadyForOffboard` 失敗時釋放輸入） | 輸入回到可花 | `wallet.bitcoin-send.failed` | 重新報價 | 是 | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | 結果未知（確認呼叫超時或報錯） | `start_send_onchain_confirmed` 已寫檢查點但返回超時 | 同所在 Bark 狀態 | `wallet.bitcoin-send.uncertain` | 先查原操作，不能新建 | 否 | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

「同上」指本表上一行。介面顯示列中的 `xxx.yyy` 為本地化文案鍵。
