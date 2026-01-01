# C1 Message send

[Back to contents](../README.md) · C · Messaging & relationships

## State diagram

GitHub renders the Mermaid block below directly; if it does not display correctly, open the SVG.

```mermaid
flowchart TD
  A(("User sends")) --> J["Send job<br/>Fetching / Uploading / Preparing"]
  J -->|"[Local] terminal error code"| F["Failed"]
  J -.- JL1(["⟲ [Session / group] retryable failure → 1 s…60 s backoff"])
  class JL1 loop
  J -->|"[User] cancel (Text / Files, no transport event yet)"| CX["Canceled"]
  J -->|"[Local] hand to transport"| S["Sending<br/>transport events + per-relay rows"]
  S -->|"[Relay] primary unit succeeded"| OK["Succeeded"]
  S -->|"[Relay] transport policy terminal"| TF["Failed<br/>manual retry"]
  J -->|"[User] delete conversation / identity"| GONE["Job deleted"]
  class OK,TF,F,CX,GONE term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C1.svg) · [Mermaid source](diagrams/C1.mmd)

## State table

| ID | State (code name) | Entry condition | Where funds / data are | UI display | What the user can do | Terminal? | Code location |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | Link-preview intent unfinished | Send job | Sending | Text / file can be canceled | No | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | Attachment intent unfinished | Send job + local attachments | Uploading | Same as above | No | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | Job exists, not yet handed to transport | Send job | Sending | Same as above | No | `delivery_queries.rs:430-432` |
| C1-04 | `Failed` (inside job) | Upload paused | Send job | Failed | Retry | No | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | Transport events created; neither business success nor terminal | `app_message_transport_events` + per-relay delivery rows | Sending | Wait | No | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | Primary unit reached business success | Delivery rows | Sent | — | Yes | `delivery_queries.rs:433-437` |
| C1-07 | `Failed` (delivery) | Transport policy judged terminal | Delivery rows | Failed | Manual retry (per `manual_retryable`) | Yes | `delivery_queries.rs:438-442` |

"Same as above" refers to the previous row of the table. In the UI column, `xxx.yyy` values are localization keys.
