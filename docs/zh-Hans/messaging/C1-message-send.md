# C1 消息发送

[返回目录](../README.md) · C · 消息与关系

## 状态图

GitHub 会直接渲染下方的 Mermaid 代码块；如果显示异常，请打开 SVG 图。

```mermaid
flowchart TD
  A(("用户发送")) --> J["发送作业<br/>Fetching / Uploading / Preparing"]
  J -->|"【本地】终态错误码"| F["Failed"]
  J -.- JL1(["⟲ 【会话 / 群】可重试失败 → 1s…60s 退避重试"])
  class JL1 loop
  J -->|"【用户】撤销（Text / Files，尚无传输事件）"| CX["已撤销"]
  J -->|"【本地】交给传输层"| S["Sending<br/>传输事件 + 每 relay 投递行"]
  S -->|"【relay】主要单元业务成功"| OK["Succeeded"]
  S -->|"【relay】传输策略判终态"| TF["Failed<br/>可手动重试"]
  J -->|"【用户】删除整个会话 / 身份"| GONE["作业删除"]
  class OK,TF,F,CX,GONE term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 图](diagrams/C1.svg) · [Mermaid 源](diagrams/C1.mmd)

## 状态表

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | 有链接预览意图未完成 | 发送作业 | 发送中 | 文本 / 文件可撤销 | 否 | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | 附件意图未完成 | 发送作业 + 本地附件 | 上传中 | 同上 | 否 | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | 作业存在，尚未交给传输层 | 发送作业 | 发送中 | 同上 | 否 | `delivery_queries.rs:430-432` |
| C1-04 | `Failed`（作业内） | 上传被暂停 | 发送作业 | 失败 | 重试 | 否 | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | 已生成传输事件，未达业务成功也未终态 | `app_message_transport_events` + 每 relay 投递行 | 发送中 | 等待 | 否 | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | 主要单元达到业务成功 | 投递行 | 已发送 | — | 是 | `delivery_queries.rs:433-437` |
| C1-07 | `Failed`（投递） | 传输策略判终态 | 投递行 | 失败 | 手动重试（视 `manual_retryable`） | 是 | `delivery_queries.rs:438-442` |

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。
