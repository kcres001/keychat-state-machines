# Keychat 状态机：状态图与状态表

[English](../en/README.md) · [Español](../es/README.md) · [Français](../fr/README.md) · [日本語](../ja/README.md) · **简体中文** · [繁體中文](../zh-Hant/README.md)

本文档描述 Keychat 中跨多步、可能需要等待或涉及资金的流程：每个流程有哪些状态、如何进入、资金或数据在哪里、界面如何显示、用户能做什么、哪些是终态，以及每条转移由谁触发。

代码基线：Keychat iOS 1.5 代码库提交 `97c69dee1`。所有行号对应该提交。

## 怎么读

### 图例

- **绿色**：终态
- **黄色**：等待用户处理或可恢复的状态
- **蓝紫菱形**：前置检查 / 判断
- **虚线圆角 ⟲**：停在原状态的循环（重试、等待、停泊），标签写明触发方
- **实线箭头**：状态转移；标签开头括号内为触发方（用户、服务器、链 / 链源、对端、mint、relay、notify、维护、启动恢复、本地）
- **虚线箭头**：仅在特定条件下发生的转移

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。

### 路径缩写

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

单元格内单独的 `:行号` 沿用同一单元格中前一个文件。

## 目录

### A · 链上与 Ark 资金

- [A1 转入 Ark、直接发送 BTC、RBF 加速](ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 从 Ark 发送 BTC 到链上地址](ark-onchain/A2-ark-to-onchain-send.md)
- [A3 出口预留](ark-onchain/A3-exit-fee-reservation.md)
- [A4 强制退出与认领](ark-onchain/A4-force-exit-and-claim.md)
- [A5 刷新与自动维护](ark-onchain/A5-refresh-and-maintenance.md)

### B · 支付与收款

- [B1 Lightning 收发（Ark）](payments/B1-lightning-ark.md)
- [B2 Cashu 收款](payments/B2-cashu-receive.md)
- [B3 聊天转账、红包与收款回执](payments/B3-chat-transfer-red-packet.md)
- [B4 消息邮资与找回](payments/B4-relay-postage-reclaim.md)
- [B5 付费通话票](payments/B5-paid-call-tickets.md)

### C · 消息与关系

- [C1 消息发送](messaging/C1-message-send.md)
- [C2 附件上传与下载](messaging/C2-attachments.md)
- [C3 好友请求与 PQXDH 升级](messaging/C3-friend-requests.md)
- [C4 群邀请与 MLS Welcome 激活](messaging/C4-group-invites-mls.md)

### D · 身份、设备与恢复

- [D1 身份删除](identity-device-recovery/D1-identity-deletion.md)
- [D2 设备激活、启动与备份恢复](identity-device-recovery/D2-device-activation-restore.md)
- [D3 钱包恢复](identity-device-recovery/D3-wallet-recovery.md)

- [全部状态（一张表）](all-states.md)
