# C3 友達リクエストと PQXDH アップグレード

[目次に戻る](../README.md) · C · メッセージと関係

## 状態図

GitHub は下の Mermaid ブロックを直接描画します。表示が崩れる場合は SVG を開いてください。

```mermaid
flowchart TD
  U(("ユーザーが友達リクエストを送信")) --> P["Pending<br/>配信再試行スケジューラ内"]
  P -->|"【リレー】受理"| PB["Published"]
  P -->|"【ユーザー】取り下げ"| C["取り下げ済み"]
  PB -->|"【ユーザー】取り下げ"| C
  P -->|"【ユーザー】同じ相手に新しいリクエスト"| SP["置き換え済み"]
  PB -->|"【相手】承認"| OK["連絡先"]
  PB -.- PBL1(["⟲ 【相手】返信なし → 待つ"])
  class PBL1 loop
  class OK,C,SP term
  class PB user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C3.svg) · [Mermaid ソース](diagrams/C3.mmd)

## 状態表

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（送信側） | リクエストを配信再試行スケジューラに投入 | 保留中のリクエスト + プロトコル層の保留セッション | 相手を待つ | 取り下げ | いいえ | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 少なくとも 1 つのリレーが受理 | 同上 | 相手を待つ | 取り下げ | いいえ | `A/types.rs:3512` |
| C3-03 | 相手が承認 | 相手の返信を受信 | セッション確立 | 連絡先 | — | はい | — |
| C3-04 | 取り下げ済み | ユーザーが取り下げ。または同じ相手への新しいリクエストが置き換え | プロトコルの保留セッションとトランスポートを消去 | — | — | はい | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH アップグレードの再試行 | ユーザーが手動で実行 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` ごとに別の表示 | 後で再試行 | 呼び出しごとに返る | `A/types.rs:533-540` |

「同上」は表の直前の行を指します。UI 列の `xxx.yyy` はローカライズキーです。
