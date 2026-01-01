# B3 Transferencias en chat, sobres rojos y recibos

[Volver al índice](../README.md) · B · Pagos y cobros

## Diagrama de estados

GitHub muestra directamente el bloque Mermaid de abajo; si no se ve bien, abra el SVG.

### B3a Remitente: traspaso de cartera (uno por sala e identidad)

```mermaid
flowchart TD
  H0(("El usuario confirma el importe")) --> P0["Preparing<br/>no se han movido fondos"]
  P0 -->|"[Mint] saga Cashu iniciada"| PC["Preparing · con operación del proveedor"]
  PC -->|"[Usuario] continuar: recuperar el token de la saga"| PD["Prepared<br/>los fondos salieron de la cartera"]
  PC -.-|"[Usuario] al eliminar el mint se borra el registro"| DEL["Registro de traspaso borrado"]
  P0 -->|"[Usuario] transferencia Ark"| PA["Preparing · resultado de Ark desconocido"]
  PA -->|"[Usuario] conciliar por id de tx"| PD
  PA -->|"[Usuario] abandonar tras confirmar que no se envió"| DEL
  PD -->|"[Usuario] entregar el mensaje al emisor"| DONE["Traspaso hecho"]
  class DONE,DEL term
  class PC,PA user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/B3a.svg) · [Fuente Mermaid](diagrams/B3a.mmd)

### B3b Lado del mensaje: transferencia Cashu / sobre rojo (DM de Signal)

```mermaid
flowchart TD
  M0(("Mensaje de pago recibido")) --> AV["Available"]
  AV -->|"[Destinatario] reclamar: primero se guarda la fase"| ACC["Accepting<br/>reclamando, reintentable"]
  ACC -->|"[Mint] canjeado, recibo enviado"| ACD["Accepted"]
  AV -->|"[Otro participante] recibo: reclamado / reembolsado"| ACD
  ACC -->|"[Otro participante] recibo: reclamado / reembolsado"| ACD
  ACC -.- ACCL1(["⟲ [Mint] canje fallido → reintento"])
  class ACCL1 loop
  AV -->|"[Remitente] reembolso tras caducar el sobre rojo"| RF["Refunding<br/>la interfaz lo reanuda sola"]
  RF -->|"[Mint] autocanjeado, recibo enviado"| RFD["Refunded"]
  RF -->|"[Mint] ya reclamado"| SU["SpentUnattributed (remitente)"]
  class ACD,RFD,SU term
  class ACC,RF user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/B3b.svg) · [Fuente Mermaid](diagrams/B3b.mmd)

## Tabla de estados

### Remitente: traspaso de cartera (como mucho uno por sala)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`, sin operación del proveedor | El usuario confirma el importe | No se han movido fondos | — | Continuar | No | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`, con operación del proveedor (Cashu) | Saga de envío Cashu iniciada | Saga en el mint; el token puede existir ya | `wallet.chat.previous-cashu-interrupted` | «Continuar»: recuperar el token de la saga | No | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`, resultado de Ark sin resolver | Resultado de la transferencia Ark desconocido | Puede haber salido ya de la cuenta | `wallet.chat.previous-wallet-transfer-interrupted` | Conciliar por id de tx; o abandonar tras confirmar que no se envió nada | No | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | Token (Cashu) o id de tx (Ark) obtenido | Los fondos salieron de la cartera; token en la saga local | `wallet.chat.wallet-transfer-complete` (se puede reanudar el traspaso del mensaje) | Seguir enviando el mensaje | No | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | Borrado (hecho) | Mensaje entregado al emisor | Registro de traspaso borrado | — | — | Sí | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |

### Lado del mensaje: transferencia Cashu / sobre rojo (solo DM de Signal)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B3-06 | ninguno → `Pending` / `Available` | Recibido o enviado | — | Reclamable | El destinatario reclama; el remitente no puede reclamar (sobre rojo) | No | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | El destinatario pulsa reclamar; la fase se guarda antes de canjear | — | Reclamando, `can_retry` | Reintentar | No | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | Canjeado y recibo enviado; o recibo del otro participante recibido | — | Reclamado | Reenviar el recibo si falló | Sí | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | El remitente inicia el reembolso tras caducar el sobre rojo | — | `chat.payment.shared.refunding` / `continue-refund` | Continuar el reembolso (la interfaz lo reanuda sola) | No | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | Reembolso completo | — | Reembolsado | Reenviar el recibo si falló | Sí | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed` (observado por el remitente) | El remitente ve el token gastado sin recibo | — | `payment_spent_without_receipt` | — | Sí (proyección) | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

«Igual que arriba» se refiere a la fila anterior de la tabla. En la columna de interfaz, los valores `xxx.yyy` son claves de localización.
