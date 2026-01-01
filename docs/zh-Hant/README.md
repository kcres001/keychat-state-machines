# Keychat 狀態機：狀態圖與狀態表

[English](../en/README.md) · [Español](../es/README.md) · [Français](../fr/README.md) · [日本語](../ja/README.md) · [简体中文](../zh-Hans/README.md) · **繁體中文**

本文件描述 Keychat 中跨多步、可能需要等待或涉及資金的流程：每個流程有哪些狀態、如何進入、資金或資料在哪裡、介面如何顯示、使用者能做什麼、哪些是終態，以及每條轉移由誰觸發。

程式碼基線：Keychat iOS 1.5 程式碼庫提交 `97c69dee1`。所有行號對應該提交。

## 怎麼讀

### 圖例

- **綠色**：終態
- **黃色**：等待使用者處理或可恢復的狀態
- **藍紫菱形**：前置檢查 / 判斷
- **虛線圓角 ⟲**：停在原狀態的迴圈（重試、等待、停泊），標籤寫明觸發方
- **實線箭頭**：狀態轉移；標籤開頭括號內為觸發方（使用者、伺服器、鏈 / 鏈源、對端、mint、relay、notify、維護、啟動恢復、本地）
- **虛線箭頭**：僅在特定條件下發生的轉移

「同上」指本表上一行。介面顯示列中的 `xxx.yyy` 為本地化文案鍵。

### 路徑縮寫

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

單元格內單獨的 `:行號` 沿用同一單元格中前一個檔案。

## 目錄

### A · 鏈上與 Ark 資金

- [A1 轉入 Ark、直接傳送 BTC、RBF 加速](ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 從 Ark 傳送 BTC 到鏈上地址](ark-onchain/A2-ark-to-onchain-send.md)
- [A3 出口預留](ark-onchain/A3-exit-fee-reservation.md)
- [A4 強制退出與認領](ark-onchain/A4-force-exit-and-claim.md)
- [A5 重新整理與自動維護](ark-onchain/A5-refresh-and-maintenance.md)

### B · 支付與收款

- [B1 Lightning 收發（Ark）](payments/B1-lightning-ark.md)
- [B2 Cashu 收款](payments/B2-cashu-receive.md)
- [B3 聊天轉賬、紅包與收款回執](payments/B3-chat-transfer-red-packet.md)
- [B4 訊息郵資與找回](payments/B4-relay-postage-reclaim.md)
- [B5 付費通話票](payments/B5-paid-call-tickets.md)

### C · 訊息與關係

- [C1 訊息傳送](messaging/C1-message-send.md)
- [C2 附件上傳與下載](messaging/C2-attachments.md)
- [C3 好友請求與 PQXDH 升級](messaging/C3-friend-requests.md)
- [C4 群邀請與 MLS Welcome 啟用](messaging/C4-group-invites-mls.md)

### D · 身份、裝置與恢復

- [D1 身份刪除](identity-device-recovery/D1-identity-deletion.md)
- [D2 裝置啟用、啟動與備份恢復](identity-device-recovery/D2-device-activation-restore.md)
- [D3 錢包恢復](identity-device-recovery/D3-wallet-recovery.md)

- [全部狀態（一張表）](all-states.md)
