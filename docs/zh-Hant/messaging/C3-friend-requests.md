# C3 好友請求與 PQXDH 升級

[返回目錄](../README.md) · C · 訊息與關係

## 狀態圖

GitHub 會直接渲染下方的 Mermaid 程式碼塊；如果顯示異常，請開啟 SVG 圖。

```mermaid
flowchart TD
  U(("使用者發起好友請求")) --> P["Pending<br/>進入投遞重試排程"]
  P -->|"【relay】接受"| PB["Published"]
  P -->|"【使用者】撤回"| C["已撤回"]
  PB -->|"【使用者】撤回"| C
  P -->|"【使用者】向同一對端發新請求"| SP["被取代"]
  PB -->|"【對端】接受"| OK["成為聯絡人"]
  PB -.- PBL1(["⟲ 【對端】未回覆 → 等待"])
  class PBL1 loop
  class OK,C,SP term
  class PB user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 圖](diagrams/C3.svg) · [Mermaid 源](diagrams/C3.mmd)

## 狀態表

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（發出方） | 請求已提交入投遞重試排程 | 待發請求 + 協議層待定會話 | 等待對方 | 撤回請求 | 否 | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 至少一處 relay 接受 | 同上 | 等待對方 | 撤回請求 | 否 | `A/types.rs:3512` |
| C3-03 | 被對方接受 | 收到對方回覆 | 會話建立 | 聯絡人 | — | 是 | — |
| C3-04 | 撤回 | 使用者撤回；或同一對端的新請求取代舊請求 | 清除協議層待定會話與傳輸 | — | — | 是 | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH 升級重試 | 使用者手動觸發 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` 分別提示 | 稍後重試 | 每次呼叫即返回 | `A/types.rs:533-540` |

「同上」指本表上一行。介面顯示列中的 `xxx.yyy` 為本地化文案鍵。
