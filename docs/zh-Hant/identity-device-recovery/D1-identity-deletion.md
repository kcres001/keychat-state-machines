# D1 身份刪除

[返回目錄](../README.md) · D · 身份、裝置與恢復

## 狀態圖

GitHub 會直接渲染下方的 Mermaid 程式碼塊；如果顯示異常，請開啟 SVG 圖。

```mermaid
flowchart TD
  U(("使用者確認刪除身份")) --> P1["PlatformCredentialPending<br/>寫入刪除計劃，結束進行中通話"]
  P1 -->|"【使用者】回滾（僅此階段）"| RB["刪除取消"]
  P1 -->|"【平臺】確認鑰匙串憑據已刪"| P2["CoreFinalizePending"]
  P2 -->|"【本地】Core 清理完成"| DONE["刪除完成"]
  P2 -.- P2L1(["⟲ 【本地】中斷後繼續"])
  class P2L1 loop
  L(("舊版刪除流程")) --> L1["LegacyCorePending"]
  L1 --> L2["LegacyPlatformSecretPending"]
  L2 -->|"【使用者】確認舊秘密已刪"| DONE
  class DONE,RB term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 圖](diagrams/D1.svg) · [Mermaid 源](diagrams/D1.mmd)

## 狀態表

| 編號 | 狀態（程式碼名） | 進入條件 | 資金或資料在哪 | 介面顯示 | 使用者可做的事 | 是否終態 | 程式碼位置 |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | 使用者確認刪除，Core 寫入刪除計劃；結束該身份的進行中通話 | 身份仍在，新工作被擋 | — | 回滾（只此階段允許） | 否 | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | 平臺確認已刪鑰匙串憑據 | 憑據已刪，Core 資料待清 | — | 繼續（可重複） | 否 | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | 舊版刪除流程的中間點 | 同上 | — | 繼續；確認舊秘密已刪 | 否 | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | 刪除完成 | Core 清理完成，刪除行移除 | 身份消失 | — | — | 是 | — |

「同上」指本表上一行。介面顯示列中的 `xxx.yyy` 為本地化文案鍵。
