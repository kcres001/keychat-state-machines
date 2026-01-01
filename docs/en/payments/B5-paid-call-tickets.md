# B5 Paid call tickets

[Back to contents](../README.md) · B · Payments

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

```mermaid
flowchart TD
  U(("User confirms purchase")) --> PRE["preparing<br/>order written"]
  PRE -->|"[Wallet] token issue failed"| FL["failed"]
  PRE -->|"[Wallet] token issued and written"| PEN["pending<br/>token in order"]
  PEN -->|"[notify] processing"| REC["reconciling"]
  REC --> PEN
  PEN -->|"[notify] success"| SUC["succeeded<br/>pass available / in use / used"]
  REC -->|"[notify] success"| SUC
  PEN -->|"[notify] failure"| FL
  REC -->|"[notify] failure"| FL
  PEN -.- PENL1(["⟲ [User] refresh on payments page"])
  class PENL1 loop
  class SUC,FL term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/B5.svg) · [Mermaid source](diagrams/B5.mmd)

## State table

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | User confirms purchase; order row written | Token not yet issued; or issued but not yet written | Paying | Continue / cancel | No | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | Token written to the order and given to notify | Token in the order | Confirming; confirm page offers only Continue | Keep polling | No | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify confirmed; then projected by usage | Pass | Available / in use / used | Call | Yes (payment side) | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify returned failure, or wallet failed to issue a token | Token cleared | Failed | — | Yes | `rtc_tickets.rs:118-125`, `:152` |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
