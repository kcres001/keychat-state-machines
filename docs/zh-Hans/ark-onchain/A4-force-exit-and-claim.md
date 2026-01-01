# A4 强制退出与认领

[返回目录](../README.md) · A · 链上与 Ark 资金

## 状态图

GitHub 会直接渲染下方的 Mermaid 代码块；如果显示异常，请打开 SVG 图。

```mermaid
flowchart TD
  U(("用户发起强退")) --> G1{"Keychat 前置检查<br/>pending_board、pending_lightning_send、pending_exit 均为 0？"}
  G1 -->|"否"| BLK["拒绝发起"]
  G1 -->|"是"| G2{"Bark 前置检查<br/>轮次均为计划等待？刷新记录均有未 forfeit 证明？"}
  G2 -->|"否"| BLK
  G2 -->|"是：候选为可花 VTXO 与恢复持有的 VTXO"| S["Start / Processing<br/>逐层广播，共享 BTC 付 CPFP"]
  AUTO(("系统自动退出")) --> S
  S -.- SL1(["⟲ 【维护】共享 BTC 资金不足 → 等待补充"])
  class SL1 loop
  S -->|"【链】全部退出交易确认"| AD["AwaitingDelta<br/>时间锁运行中"]
  AD -->|"【链】时间锁到期"| CL["Claimable"]
  S -->|"【用户】最终交易广播前取消"| CA["Canceled<br/>VTXO 仍可花"]
  S -->|"【链下】VTXO 已被花 /【服务器】中途清扫"| GONE["VtxoAlreadySpent / VtxoSwept"]
  CL -->|"【用户】确认认领报价"| P["认领 Prepared<br/>签名交易落库，输入绑定"]
  P -->|"【用户】提交：先改状态再广播"| SB["Submitting"]
  SB -->|"【链源】广播成功"| SD["Submitted<br/>Bark：ClaimInProgress"]
  SD -->|"【链】确认并写库"| CF["Confirmed / Claimed"]
  SD -->|"【链】确认，写库失败"| CK["ConfirmationCheckpointRequired"]
  CK -->|"【用户】刷新"| CF
  class CF,GONE,BLK term
  class CA,CK,P user
  class G1,G2 gate
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG 图](diagrams/A4.svg) · [Mermaid 源](diagrams/A4.mmd)

## 状态表

### 退出（每个 VTXO 一条）

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | 用户确认强退，或系统为 board / HTLC 自动退出 | 退出链交易逐层广播，需要共享 BTC 付 CPFP | `wallet.ark.exit-in-progress`、`wallet.ark.pending-funding`、`wallet.ark.refresh-to-progress-exit` | 刷新推进；补充链上资金 | 否 | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | 全部退出交易确认 | 退出输出，时间锁未到 | `wallet.ark.claim-timelock-running` | 等待 | 否 | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | 时间锁到期 | 退出输出，仅用户可花 | `wallet.ark.claim-to-bitcoin-address` | 认领 | 否 | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | 链上出现花费该输出的内存池交易 | 认领交易在内存池 | `wallet.ark.claim-in-progress` | 等待 | 否；交易掉出内存池回到 `Claimable` | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | 花费交易确认 | 在用户地址 | — | — | 是 | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO 已在链下被花；或服务器在退出中途清扫 | 无 | — | — | 是 | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | 最终交易广播前取消 | VTXO 仍可花 | — | 重新发起 | 可恢复 | `BK/exit/models/mod.rs:48-51` |

### Keychat 认领记录（`ark_claims`，输入与签名交易一对一绑定）

| 编号 | 状态（代码名） | 进入条件 | 资金或数据在哪 | 界面显示 | 用户可做的事 | 是否终态 | 代码位置 |
|---|---|---|---|---|---|---|---|
| A4-08 | `Prepared` | 用户确认报价，签名交易落库，输入登记到 `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | 「提交原交易」 | 否 | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | 开始广播（先改状态再广播） | — | 认领中 | 等待 | 否 | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | 广播返回成功，或链源报内存池 | — | `wallet.ark.claim-in-progress-message` | 「刷新」 | 否 | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired`（投影） | 链上已确认但写库失败 | — | `wallet.ark.claim-confirmation-checkpoint-required` | 刷新 | 否 | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | 链上确认并写库 | — | 已认领 | — | 是 | `claim_store.rs:287-293` |

「同上」指本表上一行。界面显示列中的 `xxx.yyy` 为本地化文案键。
