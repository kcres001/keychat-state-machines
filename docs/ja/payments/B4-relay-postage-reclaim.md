# B4 リレー郵送料と回収

[目次に戻る](../README.md) · B · 支払いと受け取り

## 状態図

GitHub は下の Mermaid ブロックを直接描画します。表示が崩れる場合は SVG を開いてください。

```mermaid
flowchart TD
  D(("有料リレーにメッセージを配信")) -->|"【ローカル】郵送料トークンを作成"| CR["Created<br/>トークンをリレーに渡した"]
  D -->|"リレーは無料"| NR["NotRequired"]
  CR -->|"【リレー】OK"| OKD["配信成功"]
  CR -->|"【リレー】失敗 / タイムアウト / OK なし"| FD["配信失敗"]
  FD -->|"【ユーザー】詳細を開く"| AV["Available"]
  FD -->|"一意に対応する送金なし"| UA["Unavailable"]
  AV -->|"【ユーザー】回収：配信行を終了状態にし、意図を永続化"| PRC["Processing"]
  PRC -->|"【ミント】全証明が使用済み"| UA
  PRC -.- PRCL1(["⟲ 【ミント】到達不能 / 不明 → ユーザーが再試行"])
  class PRCL1 loop
  PRC -->|"【ミント】交換完了"| RC["Reclaimed"]
  class OKD,NR,UA,RC term
  class PRC user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/B4.svg) · [Mermaid ソース](diagrams/B4.mmd)

## 状態表

### 配信ごと（メッセージ × リレー）

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | リレーは郵送料を取らない | — | — | — | はい（その配信について） | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | 未払い、または支払えない | 未払い | — | — | いいえ | same as above |
| B4-03 | `Created` | 郵送料トークンを作成し、イベントとともにリレーへ送信 | トークンをリレーに渡し、ウォレット台帳に `relay_stamp` 送金を記録 | — | — | はい（その配信について） | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |

### 回収（失敗した配信行のみ）

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| B4-04 | 回収不可（結び付けなし） | 配信行の状態が failed / timeout / no_ack / terminal_failed / retry_exhausted のいずれでもない、または OK を受信済み | — | 入口を表示しない | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | 結び付けが `TokenCreated` の送金 saga 1 件にのみ対応 | トークンはまだリレーに換金されていない可能性 | 回収を表示 | 回収 | いいえ | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | 回収の意図を受理（ミント操作前に永続化）、またはミントの状態が不明 / 到達不能 | 意図 + 元の証明 | 処理中 | 再度回収 | いいえ | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | 交換完了 | ウォレットに戻った | 回収済み | — | はい | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | 全証明が使用済み（リレーが換金済み）、または一意に対応する台帳エントリなし | リレー側 | 回収不可 | — | はい | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

「同上」は表の直前の行を指します。UI 列の `xxx.yyy` はローカライズキーです。
