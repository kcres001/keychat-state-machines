# B2 Recepción Cashu

[Volver al índice](../README.md) · B · Pagos y cobros

## Diagrama de estados

GitHub muestra directamente el bloque Mermaid de abajo; si no se ve bien, abra el SVG.

```mermaid
flowchart TD
  T(("Token recibido, el usuario reclama")) -->|"mint registrado"| AC["Accepted<br/>token guardado localmente"]
  T -->|"mint no registrado"| BL["Blocked · UnknownMint"]
  BL -->|"[Usuario] añadir mint"| AC
  AC -->|"[Mantenimiento / usuario] conciliar"| RC["Reconciling"]
  RC -->|"[Mint] abonado / ya recibido / recuperado"| TOK["Terminal<br/>recibido"]
  RC -->|"[Mint] token ya gastado"| TSP["Terminal · AlreadySpent"]
  RC -.- RCL1(["⟲ [Mint] saga incompleta → seguir conciliando"])
  class RCL1 loop
  RC -->|"[Mint] error, sin id de operación del SDK"| RD["RetryDue<br/>reintento en 30 s"]
  RD -->|"[Mantenimiento] pasaron 30 s"| RC
  class TOK,TSP term
  class BL user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/B2.svg) · [Fuente Mermaid](diagrams/B2.mmd)

## Tabla de estados

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | Token recibido, mint registrado | Token al portador en el almacén local cifrado (la única copia reutilizable) | Recepción en curso | Esperar | No | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked` (`UnknownMint`) | Mint no registrado | Igual que arriba | Hace falta añadir el mint | Añadir el mint y continuar | No | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | Conciliando, o el SDK ya tiene una saga para ello | Token + saga de CDK | Procesando | Esperar | No | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue` (`RetryRequired`) | La conciliación falló sin id de operación del SDK; reintento en 30 s | Token | Procesando | Esperar | No | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal` (sin incidencia) | Se obtuvo `Credited` / `AlreadyReceived` / `Recovered` | En el saldo de CDK | Recibido | — | Sí | `receive_operations.rs:196-206` |
| B2-06 | `Terminal` (`AlreadySpent`) | El mint dice que el token ya se gastó | Ninguno | Error `CashuTokenAlreadySpent` | — | Sí | `receive_operations.rs:207-213` |

«Igual que arriba» se refiere a la fila anterior de la tabla. En la columna de interfaz, los valores `xxx.yyy` son claves de localización.
