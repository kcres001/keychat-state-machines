# D3 Wallet recovery

[Back to contents](../README.md) · D · Identity, device & recovery

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

```mermaid
flowchart TD
  U(("User restores Ark from mnemonic")) --> REG["Account registered<br/>reserve None, recovery_pending"]
  REG -->|"[Server] scan epochs"| Q{"Result"}
  Q -->|"all complete"| OK["Restored"]
  Q -->|"no server record"| NF["NotFound"]
  Q -->|"server unavailable"| UA["Unavailable"]
  Q -->|"server identity changed"| IC["IdentityChanged"]
  Q -->|"some epoch incomplete"| IN["Incomplete<br/>cooperative operations paused"]
  IN -->|"[User] recover again"| REG
  REG -->|"[User] extend key scan"| KS["KeyScan<br/>Ready → LimitReached / CandidatesMatched"]
  class OK,NF,UA,IC term
  class IN,KS user
  class Q gate
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/D3.svg) · [Mermaid source](diagrams/D3.mmd)

## State table

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | All epochs complete | Recovered VTXOs and exits | — | — | Yes | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | Server has no record of the wallet | — | — | Try another source | Yes | `types.rs:70` |
| D3-03 | `Incomplete` | Any epoch incomplete; account stays `recovery_pending`, cooperative operations refused, reservation None | Partially recovered | — | Recover again | No | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | Server unavailable / server identity changed | — | — | Retry later / judge manually | Yes (this attempt) | `types.rs:72-73` |
| D3-05 | Key scan `Ready` / `LimitReached` / `CandidatesMatched` | Scanned to limit or matched by index | Scan progress | — | Extend scan | No | `types.rs:89-108` |
| D3-06 | BTC history scan `Ready` / `LimitReached` / `UnsupportedChainSource` | Same as above (shared BTC) | Addresses discovered | — | Extend scan | No | `types.rs:441-457` |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
