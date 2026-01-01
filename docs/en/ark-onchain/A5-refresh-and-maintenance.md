# A5 Refresh and automatic maintenance

[Back to contents](../README.md) · A · On-chain & Ark funds

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

### A5a One refresh (Bark round)

```mermaid
flowchart TD
  SW["NonInteractivePending · scheduled-waiting<br/>inputs not locked"]
  SW -->|"[Chain] target height reached"| DP
  SW -->|"[User] retired for force exit"| RET["Retired"]
  SUB(("Submit delegated refresh")) --> DP["NonInteractivePending · delegated<br/>inputs locked, counted in pending_in_round"]
  DP -.- DPL1(["⟲ [Server] waiting for a round"])
  class DPL1 loop
  DP -->|"[Server] round issued"| FIN
  IS(("Interactive submit")) --> IP["InteractivePending"]
  IP -->|"[Server] round starts"| IO["InteractiveOngoing"]
  IP -->|"[Local] inputs lost, cannot shrink"| CAN["Canceled"]
  IO -->|"[Server] interaction done"| FIN["Finished<br/>awaiting funding tx confirmation"]
  IO -->|"[Server] failed"| FAIL["Failed"]
  FIN -.- FINL2(["⟲ [Chain] waiting for confirmation"])
  class FINL2 loop
  FIN -->|"[Chain] confirmed + forfeits done"| OK["Confirmed<br/>new VTXOs"]
  class OK,FAIL,CAN,RET term
  class SW user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/A5a.svg) · [Mermaid source](diagrams/A5a.mmd)

### A5b Keychat maintenance phase (per account)

```mermaid
flowchart TD
  MI["Idle / Disabled"] -->|"[Maintenance] due, plan approved"| MRd["Ready"]
  MI -->|"[Maintenance] plan needs the user"| MH["AuthorizationRequired / ManualReview / NeedsFunds"]
  MH -->|"[User] authorize / review / add funds"| MRd
  MRd -->|"[Maintenance] submit"| MRf["Refreshing"]
  MRf -->|"[Server] evidence Succeeded / Rejected"| MI
  MRf -->|"[Local] failed, no pending submission"| MRt["Retrying<br/>60 s → 30 min backoff"]
  MRt -->|"[Maintenance / network restored]"| MRd
  MRf -->|"[Local] submission outcome unknown"| MRq["RecoveryRequired<br/>recover before submitting"]
  MRq -->|"[Server] evidence arrives"| MI
  class MH,MRq user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/A5b.svg) · [Mermaid source](diagrams/A5b.mmd)

## State table

### One refresh (Bark round)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | Scheduled refresh accepted by server, waiting for target height; scheduled-waiting: yes (`is_scheduled_waiting`) | Inputs not locked; still force-exitable | — | — | No | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | Delegated refresh submitted, waiting for the server to run a round; scheduled-waiting: no | Inputs locked by the movement, counted in `pending_in_round` | — | — | No | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | Interactive round queued / in progress; scheduled-waiting: no | Same as above | — | — | No | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished` (not done) | Interaction done, waiting for round funding tx to confirm; scheduled-waiting: no | Same as above; forfeits not yet complete | — | — | No | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished` (done) → `Confirmed` | Funding tx confirmed and forfeits complete; scheduled-waiting: — | New VTXOs | — | — | Yes | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | Round failed, user canceled, or inputs lost and cannot shrink; scheduled-waiting: — | Inputs released | — | — | Yes | `BK/round/mod.rs:793-813` |

### Keychat maintenance phase (per account)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| A5-07 | `Idle` / `Disabled` | No refresh needed / auto refresh off | — | `disabled-title` | Enable, or refresh manually | No (resting) | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | Plan approved / submitted | — | — | Wait | No | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | Failure backoff 60 s → 30 min | — | — | Wait | No | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | Submission outcome unknown; recovery first | — | — | Wait (re-checked every `URGENT_CHECK_SECONDS`) | No | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | Needs user fee authorization, manual review, or more funds | — | `authorization-*`, `funds-title`, etc. | Authorize, review, or add funds | No (awaiting user) | `W/ark/maintenance.rs:35-62` |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
