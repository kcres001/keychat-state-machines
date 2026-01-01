# Keychat の状態機械：状態図と状態表

[English](../en/README.md) · [Español](../es/README.md) · [Français](../fr/README.md) · **日本語** · [简体中文](../zh-Hans/README.md) · [繁體中文](../zh-Hant/README.md)

本ドキュメントは、Keychat のうち複数の段階を持ち、待機が生じたり資金を伴ったりするフローを記述します。各フローの状態、各状態に入る条件、資金やデータの所在、UI の表示、ユーザーができること、終了状態かどうか、そして各遷移を誰が引き起こすかを示します。

コードの基準：Keychat iOS 1.5 コードベース、コミット `97c69dee1`。行番号はすべてこのコミットに対応します。

## 読み方

### 凡例

- **緑**：終了状態
- **黄**：ユーザーの対応待ち、または復旧可能
- **青紫のひし形**：事前チェック / 判断
- **破線の角丸 ⟲**：同じ状態にとどまるループ（再試行、待機、保留）。ラベルに契機を記載
- **実線の矢印**：状態遷移。ラベル冒頭の括弧内が契機（ユーザー、サーバー、チェーン / チェーンソース、相手、ミント、リレー、notify、メンテナンス、起動時復旧、ローカル）
- **破線の矢印**：特定の条件でのみ起きる遷移

「同上」は表の直前の行を指します。UI 列の `xxx.yyy` はローカライズキーです。

### パスの略記

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

セル内の単独の `:行番号` は、同じセル内の直前のファイルを指します。

## 目次

### A · オンチェーンと Ark の資金

- [A1 Ark への入金、BTC の直接送金、RBF による加速](ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 Ark からオンチェーンアドレスへの BTC 送金](ark-onchain/A2-ark-to-onchain-send.md)
- [A3 出口手数料の予約](ark-onchain/A3-exit-fee-reservation.md)
- [A4 強制退出と請求](ark-onchain/A4-force-exit-and-claim.md)
- [A5 リフレッシュと自動メンテナンス](ark-onchain/A5-refresh-and-maintenance.md)

### B · 支払いと受け取り

- [B1 Lightning の送受信（Ark）](payments/B1-lightning-ark.md)
- [B2 Cashu の受け取り](payments/B2-cashu-receive.md)
- [B3 チャット送金、紅包、受領通知](payments/B3-chat-transfer-red-packet.md)
- [B4 リレー郵送料と回収](payments/B4-relay-postage-reclaim.md)
- [B5 有料通話チケット](payments/B5-paid-call-tickets.md)

### C · メッセージと関係

- [C1 メッセージ送信](messaging/C1-message-send.md)
- [C2 添付ファイルのアップロードとダウンロード](messaging/C2-attachments.md)
- [C3 友達リクエストと PQXDH アップグレード](messaging/C3-friend-requests.md)
- [C4 グループ招待と MLS Welcome の有効化](messaging/C4-group-invites-mls.md)

### D · アイデンティティ、デバイス、復旧

- [D1 アイデンティティの削除](identity-device-recovery/D1-identity-deletion.md)
- [D2 デバイスの有効化、起動、バックアップ復元](identity-device-recovery/D2-device-activation-restore.md)
- [D3 ウォレットの復旧](identity-device-recovery/D3-wallet-recovery.md)

- [全状態（1 つの表）](all-states.md)
