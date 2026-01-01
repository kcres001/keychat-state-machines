# B2 Cashu 收款

[返回目录](../README.md) · B · 支付与收款

## 状态图

GitHub 会直接渲染下方的 Mermaid 代码块；如果显示异常，请打开 SVG 图。

```mermaid
flowchart TD
  T(("收到 token，用户领取")) -->|"mint 已注册"| AC["Accepted<br/>token 存本地"]
  T -->|"mint 未注册"| BL["Blocked · UnknownMint"]
  BL -->|"【用户】添加 mint"| AC
  AC -->|"【维护 / 用户】对账"| RC["Reconciling"]
  RC -->|"【mint】入账 / 已领过 / 已恢复"| TOK["Terminal<br/>已到账"]
  RC -->|"【mint】token 已被花"| TSP["Terminal · AlreadySpent"]
  RC -.- RCL1(["⟲ 【mint】saga 未完成 → 继续对账"])
  class RCL1 loop
  RC -->|"【mint】出错，无 SDK 操作 ID"| RD["RetryDue<br/>30 秒后再试"]
  RD -->|"【维护】30 秒到"| RC
  class TOK,TSP term
  class BL user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 图](diagrams/B2.svg) · [Mermaid 源](diagrams/B2.mmd)

## 状态表

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | 收到 token，mint 已注册 | bearer token 存于本地加密库（唯一可重放副本） | 收款处理中 | 等待 | 否 | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked`（`UnknownMint`） | mint 未注册 | 同上 | 需添加 mint | 添加 mint 后继续 | 否 | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | 正在对账，或 SDK 已有该操作的 saga | token + CDK saga | 处理中 | 等待 | 否 | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue`（`RetryRequired`） | 对账出错且没有 SDK 操作 ID，30 秒后再试 | token | 处理中 | 等待 | 否 | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal`（issue 为空） | 收到 `Credited` / `AlreadyReceived` / `Recovered` | 已入 CDK 余额 | 已到账 | — | 是 | `receive_operations.rs:196-206` |
| B2-06 | `Terminal`（`AlreadySpent`） | mint 报 token 已被花 | 无 | `CashuTokenAlreadySpent` 报错 | — | 是 | `receive_operations.rs:207-213` |

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。
