# A5 リフレッシュと自動メンテナンス

[目次に戻る](../README.md) · A · オンチェーンと Ark の資金

## 状態図

GitHub は下の Mermaid ブロックを直接描画します。表示が崩れる場合は SVG を開いてください。

### A5a リフレッシュ 1 件（Bark ラウンド）

```mermaid
flowchart TD
  SW["NonInteractivePending · 予定待機<br/>入力ロックなし"]
  SW -->|"【チェーン】目標高さに到達"| DP
  SW -->|"【ユーザー】強制退出のため撤回"| RET["撤回"]
  SUB(("委任リフレッシュを提出")) --> DP["NonInteractivePending · 委任<br/>入力ロック、pending_in_round に計上"]
  DP -.- DPL1(["⟲ 【サーバー】ラウンド待ち"])
  class DPL1 loop
  DP -->|"【サーバー】ラウンド発行"| FIN
  IS(("対話型の提出")) --> IP["InteractivePending"]
  IP -->|"【サーバー】ラウンド開始"| IO["InteractiveOngoing"]
  IP -->|"【ローカル】入力喪失、縮小不可"| CAN["Canceled"]
  IO -->|"【サーバー】対話完了"| FIN["Finished<br/>資金 tx の確定待ち"]
  IO -->|"【サーバー】失敗"| FAIL["Failed"]
  FIN -.- FINL2(["⟲ 【チェーン】確定待ち"])
  class FINL2 loop
  FIN -->|"【チェーン】確定 + forfeit 完了"| OK["Confirmed<br/>新しい VTXO"]
  class OK,FAIL,CAN,RET term
  class SW user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/A5a.svg) · [Mermaid ソース](diagrams/A5a.mmd)

### A5b Keychat のメンテナンス段階（アカウントごと）

```mermaid
flowchart TD
  MI["Idle / Disabled"] -->|"【メンテナンス】期限到来、計画承認"| MRd["Ready"]
  MI -->|"【メンテナンス】計画にユーザー対応が必要"| MH["AuthorizationRequired / ManualReview / NeedsFunds"]
  MH -->|"【ユーザー】承認 / 確認 / 資金追加"| MRd
  MRd -->|"【メンテナンス】提出"| MRf["Refreshing"]
  MRf -->|"【サーバー】証拠 Succeeded / Rejected"| MI
  MRf -->|"【ローカル】失敗、未決の提出なし"| MRt["Retrying<br/>60 秒 → 30 分のバックオフ"]
  MRt -->|"【メンテナンス / ネットワーク復帰】"| MRd
  MRf -->|"【ローカル】提出結果不明"| MRq["RecoveryRequired<br/>提出前に復旧"]
  MRq -->|"【サーバー】証拠が届く"| MI
  class MH,MRq user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/A5b.svg) · [Mermaid ソース](diagrams/A5b.mmd)

## 状態表

### リフレッシュ 1 件（Bark ラウンド）

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | 予定リフレッシュがサーバーに受理され、目標高さを待つ。予定待機：はい（`is_scheduled_waiting`） | 入力はロックされていない。強制退出可能 | — | — | いいえ | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | 委任リフレッシュを提出し、サーバーのラウンドを待つ。予定待機：いいえ | 入力は movement によりロック、`pending_in_round` に計上 | — | — | いいえ | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | 対話型ラウンドの待機中 / 進行中。予定待機：いいえ | 同上 | — | — | いいえ | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished`（未完了） | 対話完了、ラウンドの資金 tx の確定待ち。予定待機：いいえ | 同上。forfeit は未完了 | — | — | いいえ | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished`（完了）→ `Confirmed` | 資金 tx 確定、forfeit 完了。予定待機：— | 新しい VTXO | — | — | はい | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | ラウンド失敗、ユーザーがキャンセル、または入力喪失で縮小不可。予定待機：— | 入力を解放 | — | — | はい | `BK/round/mod.rs:793-813` |

### Keychat のメンテナンス段階（アカウントごと）

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| A5-07 | `Idle` / `Disabled` | リフレッシュ不要 / 自動リフレッシュ無効 | — | `disabled-title` | 有効にする、または手動でリフレッシュ | いいえ（常駐） | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | 計画承認済み / 提出済み | — | — | 待つ | いいえ | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | 失敗時のバックオフ 60 秒 → 30 分 | — | — | 待つ | いいえ | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | 提出結果不明。先に復旧 | — | — | 待つ（`URGENT_CHECK_SECONDS` ごとに再確認） | いいえ | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | ユーザーによる手数料承認、手動確認、または資金追加が必要 | — | `authorization-*`、`funds-title` など | 承認、確認、または資金追加 | いいえ（ユーザー待ち） | `W/ark/maintenance.rs:35-62` |

「同上」は表の直前の行を指します。UI 列の `xxx.yyy` はローカライズキーです。
