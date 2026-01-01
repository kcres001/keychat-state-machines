# C2 Attachment upload and download

[Back to contents](../README.md) · C · Messaging & relationships

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

### C2a Download

```mermaid
flowchart TD
  R["Remote"] -->|"[User] download"| DG["Downloading"]
  DG -->|"[Media server] done"| DD["Downloaded"]
  DG -->|"[Media server] failed"| DF["Failed"]
  DG -->|"[Startup recovery] interrupted"| DF
  DF -->|"[User] retry"| DG
  DD -.->|"local file removed"| R
  class DD term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C2a.svg) · [Mermaid source](diagrams/C2a.mmd)

### C2b Upload (part of the send job)

```mermaid
flowchart TD
  UP["Uploading"] -->|"[Media server] done"| UOK["Handed to sender"]
  UP -->|"[Media server] failed"| PS["Paused"]
  PS -->|"[User] retry (retryable errors)"| UP
  PS -->|"[User] cancel message"| UC["Canceled"]
  class UOK,UC term
  class PS user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C2b.svg) · [Mermaid source](diagrams/C2b.mmd)

## State table

### Download

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | Not downloaded | Remote URL + key | Downloadable | Download | No | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | Download started (`transfer_state=1`) | Temp file | Downloading | Wait | No | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` and local file exists | Local file | Openable | — | Yes (back to Remote projection if the file is gone) | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | Download failed, or interruption found at startup / runtime restart | `last_error_code` | Failed | Retry | No | `attachments.rs:462-469`, `:472-492` |

### Upload (part of the send job)

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C2-05 | Uploading | Attachment intent unfinished | — | `Uploading` | Text / file messages can be canceled | No | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | Paused (failure inside job) | Upload failure writes `upload_state='paused'` | — | `Failed` | Retry allowed for retryable errors; not for missing source, invalid intent, unsafe target | No | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
