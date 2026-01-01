# D1 Identity deletion

[Back to contents](../README.md) · D · Identity, device & recovery

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

```mermaid
flowchart TD
  U(("User confirms identity deletion")) --> P1["PlatformCredentialPending<br/>deletion plan written, active calls ended"]
  P1 -->|"[User] roll back (this phase only)"| RB["Deletion canceled"]
  P1 -->|"[Platform] keychain credentials deleted"| P2["CoreFinalizePending"]
  P2 -->|"[Local] Core cleanup done"| DONE["Deletion complete"]
  P2 -.- P2L1(["⟲ [Local] resume after interruption"])
  class P2L1 loop
  L(("Legacy deletion flow")) --> L1["LegacyCorePending"]
  L1 --> L2["LegacyPlatformSecretPending"]
  L2 -->|"[User] confirm legacy secret deleted"| DONE
  class DONE,RB term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/D1.svg) · [Mermaid source](diagrams/D1.mmd)

## State table

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | User confirms deletion; Core writes deletion plan; active calls of the identity ended | Identity still present; new work blocked | — | Roll back (only in this phase) | No | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | Platform confirms keychain credentials deleted | Credentials deleted; Core data pending cleanup | — | Continue (repeatable) | No | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Intermediate points of the legacy deletion flow | Same as above | — | Continue; confirm legacy secret deleted | No | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Deletion complete | Core cleanup done; deletion row removed | Identity gone | — | — | Yes | — |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
