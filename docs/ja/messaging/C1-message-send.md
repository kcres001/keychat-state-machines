# C1 メッセージ送信

[目次に戻る](../README.md) · C · メッセージと関係

## 状態図

GitHub は下の Mermaid ブロックを直接描画します。表示が崩れる場合は SVG を開いてください。

```mermaid
flowchart TD
  A(("ユーザーが送信")) --> J["送信ジョブ<br/>Fetching / Uploading / Preparing"]
  J -->|"【ローカル】終了エラーコード"| F["Failed"]
  J -.- JL1(["⟲ 【セッション / グループ】再試行可能な失敗 → 1 秒…60 秒のバックオフ"])
  class JL1 loop
  J -->|"【ユーザー】取り消し（Text / Files、トランスポートイベントなし）"| CX["取り消し済み"]
  J -->|"【ローカル】トランスポートに渡す"| S["Sending<br/>トランスポートイベント + リレーごとの行"]
  S -->|"【リレー】主要ユニット成功"| OK["Succeeded"]
  S -->|"【リレー】トランスポートポリシーで終了"| TF["Failed<br/>手動再試行"]
  J -->|"【ユーザー】会話 / アイデンティティを削除"| GONE["ジョブ削除"]
  class OK,TF,F,CX,GONE term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C1.svg) · [Mermaid ソース](diagrams/C1.mmd)

## 状態表

| ID | 状態（コード名） | 遷移条件 | 資金・データの所在 | UI 表示 | ユーザーができること | 終了状態か | コード位置 |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | リンクプレビューの意図が未完了 | 送信ジョブ | 送信中 | テキスト / ファイルは取り消し可能 | いいえ | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | 添付ファイルの意図が未完了 | 送信ジョブ + ローカル添付 | アップロード中 | 同上 | いいえ | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | ジョブあり、まだトランスポートに渡していない | 送信ジョブ | 送信中 | 同上 | いいえ | `delivery_queries.rs:430-432` |
| C1-04 | `Failed`（ジョブ内） | アップロード一時停止 | 送信ジョブ | 失敗 | 再試行 | いいえ | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | トランスポートイベントを作成。業務上の成功でも終了でもない | `app_message_transport_events` + リレーごとの配信行 | 送信中 | 待つ | いいえ | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | 主要ユニットが業務上の成功に到達 | 配信行 | 送信済み | — | はい | `delivery_queries.rs:433-437` |
| C1-07 | `Failed`（配信） | トランスポートポリシーが終了と判定 | 配信行 | 失敗 | 手動再試行（`manual_retryable` に従う） | はい | `delivery_queries.rs:438-442` |

「同上」は表の直前の行を指します。UI 列の `xxx.yyy` はローカライズキーです。
