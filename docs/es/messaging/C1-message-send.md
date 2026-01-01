# C1 Envío de mensajes

[Volver al índice](../README.md) · C · Mensajería y relaciones

## Diagrama de estados

GitHub muestra directamente el bloque Mermaid de abajo; si no se ve bien, abra el SVG.

```mermaid
flowchart TD
  A(("El usuario envía")) --> J["Trabajo de envío<br/>Fetching / Uploading / Preparing"]
  J -->|"[Local] código de error final"| F["Failed"]
  J -.- JL1(["⟲ [Sesión / grupo] fallo reintentable → espera de 1 s…60 s"])
  class JL1 loop
  J -->|"[Usuario] cancelar (Text / Files, aún sin evento de transporte)"| CX["Cancelado"]
  J -->|"[Local] entregar al transporte"| S["Sending<br/>eventos de transporte + filas por relay"]
  S -->|"[Relay] la unidad principal tuvo éxito"| OK["Succeeded"]
  S -->|"[Relay] final según la política de transporte"| TF["Failed<br/>reintento manual"]
  J -->|"[Usuario] borrar conversación / identidad"| GONE["Trabajo borrado"]
  class OK,TF,F,CX,GONE term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C1.svg) · [Fuente Mermaid](diagrams/C1.mmd)

## Tabla de estados

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | Intención de vista previa de enlace sin terminar | Trabajo de envío | Enviando | Texto / archivo se pueden cancelar | No | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | Intención de adjunto sin terminar | Trabajo de envío + adjuntos locales | Subiendo | Igual que arriba | No | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | Existe el trabajo, aún no entregado al transporte | Trabajo de envío | Enviando | Igual que arriba | No | `delivery_queries.rs:430-432` |
| C1-04 | `Failed` (dentro del trabajo) | Subida en pausa | Trabajo de envío | Fallido | Reintentar | No | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | Eventos de transporte creados; ni éxito de negocio ni final | `app_message_transport_events` + filas de entrega por relay | Enviando | Esperar | No | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | La unidad principal alcanzó el éxito de negocio | Filas de entrega | Enviado | — | Sí | `delivery_queries.rs:433-437` |
| C1-07 | `Failed` (entrega) | La política de transporte lo declara final | Filas de entrega | Fallido | Reintento manual (según `manual_retryable`) | Sí | `delivery_queries.rs:438-442` |

«Igual que arriba» se refiere a la fila anterior de la tabla. En la columna de interfaz, los valores `xxx.yyy` son claves de localización.
