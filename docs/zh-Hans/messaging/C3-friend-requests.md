# C3 好友请求与 PQXDH 升级

[返回目录](../README.md) · C · 消息与关系

## 状态图

GitHub 会直接渲染下方的 Mermaid 代码块；如果显示异常，请打开 SVG 图。

```mermaid
flowchart TD
  U(("用户发起好友请求")) --> P["Pending<br/>进入投递重试调度"]
  P -->|"【relay】接受"| PB["Published"]
  P -->|"【用户】撤回"| C["已撤回"]
  PB -->|"【用户】撤回"| C
  P -->|"【用户】向同一对端发新请求"| SP["被取代"]
  PB -->|"【对端】接受"| OK["成为联系人"]
  PB -.- PBL1(["⟲ 【对端】未回复 → 等待"])
  class PBL1 loop
  class OK,C,SP term
  class PB user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 图](diagrams/C3.svg) · [Mermaid 源](diagrams/C3.mmd)

## 状态表

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending`（发出方） | 请求已提交入投递重试调度 | 待发请求 + 协议层待定会话 | 等待对方 | 撤回请求 | 否 | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | 至少一处 relay 接受 | 同上 | 等待对方 | 撤回请求 | 否 | `A/types.rs:3512` |
| C3-03 | 被对方接受 | 收到对方回复 | 会话建立 | 联系人 | — | 是 | — |
| C3-04 | 撤回 | 用户撤回；或同一对端的新请求取代旧请求 | 清除协议层待定会话与传输 | — | — | 是 | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | PQXDH 升级重试 | 用户手动触发 | — | `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` 分别提示 | 稍后重试 | 每次调用即返回 | `A/types.rs:533-540` |

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。
