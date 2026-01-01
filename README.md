# Keychat state machines: diagrams and state tables

**English** · [Español](docs/es/README.md) · [Français](docs/fr/README.md) · [日本語](docs/ja/README.md) · [简体中文](docs/zh-Hans/README.md) · [繁體中文](docs/zh-Hant/README.md)

This document describes the multi-step flows in Keychat that may wait or involve funds: the states of each flow, how each state is entered, where funds or data are, what the UI shows, what the user can do, which states are terminal, and who triggers each transition.

Code baseline: Keychat iOS 1.5 codebase, commit `97c69dee1`. All line numbers refer to that commit.

## How to read

### Legend

- **Green**: Terminal state
- **Yellow**: Awaiting the user, or recoverable
- **Blue-violet diamond**: Precheck / decision
- **Dashed rounded ⟲**: Loop that stays in the same state (retry, wait, park); the label names the trigger
- **Solid arrow**: Transition; the bracketed prefix names the trigger (user, server, chain / chain source, peer, mint, relay, notify, maintenance, startup recovery, local)
- **Dashed arrow**: Transition that happens only under a specific condition

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.

### Path abbreviations

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

A bare `:line` in a cell refers to the preceding file in the same cell.

## Contents

### A · On-chain & Ark funds

- [A1 Onboard to Ark, direct BTC send, RBF acceleration](docs/en/ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 Sending BTC from Ark to an on-chain address](docs/en/ark-onchain/A2-ark-to-onchain-send.md)
- [A3 Exit fee reservation](docs/en/ark-onchain/A3-exit-fee-reservation.md)
- [A4 Force exit and claim](docs/en/ark-onchain/A4-force-exit-and-claim.md)
- [A5 Refresh and automatic maintenance](docs/en/ark-onchain/A5-refresh-and-maintenance.md)

### B · Payments

- [B1 Lightning send and receive (Ark)](docs/en/payments/B1-lightning-ark.md)
- [B2 Cashu receive](docs/en/payments/B2-cashu-receive.md)
- [B3 Chat transfers, red packets and receipts](docs/en/payments/B3-chat-transfer-red-packet.md)
- [B4 Relay postage and reclaim](docs/en/payments/B4-relay-postage-reclaim.md)
- [B5 Paid call tickets](docs/en/payments/B5-paid-call-tickets.md)

### C · Messaging & relationships

- [C1 Message send](docs/en/messaging/C1-message-send.md)
- [C2 Attachment upload and download](docs/en/messaging/C2-attachments.md)
- [C3 Friend requests and PQXDH upgrade](docs/en/messaging/C3-friend-requests.md)
- [C4 Group invites and MLS Welcome activation](docs/en/messaging/C4-group-invites-mls.md)

### D · Identity, device & recovery

- [D1 Identity deletion](docs/en/identity-device-recovery/D1-identity-deletion.md)
- [D2 Device activation, startup and backup restore](docs/en/identity-device-recovery/D2-device-activation-restore.md)
- [D3 Wallet recovery](docs/en/identity-device-recovery/D3-wallet-recovery.md)

- [All states (single table)](docs/en/all-states.md)
