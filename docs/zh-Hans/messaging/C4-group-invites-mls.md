# C4 群邀请与 MLS Welcome 激活

[返回目录](../README.md) · C · 消息与关系

## 状态图

GitHub 会直接渲染下方的 Mermaid 代码块；如果显示异常，请打开 SVG 图。

```mermaid
flowchart TD
  U(("发出 / 收到邀请")) --> P["pending"]
  P -->|"【relay】投递失败"| F["failed"]
  F -->|"【MLS 发出方】重试投递"| P
  P -->|"【用户】接受"| A["accepted<br/>Welcome：PendingAcknowledgement"]
  A -->|"【本地】发出激活控制消息"| FN["finalizing"]
  FN -->|"【relay OK / 对端认证证据】"| EN["enabled<br/>Welcome：Active"]
  A -.- AL1(["⟲ 【本地】激活重试"])
  class AL1 loop
  P -->|"【用户】拒绝"| DC["declined"]
  P -->|"【用户】撤回"| CN["canceled"]
  P -->|"【时间】过期"| EX["expired"]
  P -->|"【用户】删除"| RP["removePending"]
  RP -->|"【恢复】确认删除"| DC
  class EN,DC,CN,EX term
  class F user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 图](diagrams/C4.svg) · [Mermaid 源](diagrams/C4.mmd)

## 状态表

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | 发出或收到邀请；投递重试回到待定 | — | — | 接受 / 拒绝 / 撤回 | 否 | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | 投递失败、决定处理失败 | — | — | MLS 发出方可重试投递（`group_invite_lifecycle_allows_delivery_retry`） | 否 | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | 接受后等待成员激活控制消息落地 | — | `PendingAcknowledgement` | 等待 | 否 | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | 收到认证的激活证据或 relay OK | — | `Active` | — | 是 | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | 拒绝、撤回、过期 | — | — | — | 是 | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | 删除过程中待确认 | — | — | — | 否 | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。
