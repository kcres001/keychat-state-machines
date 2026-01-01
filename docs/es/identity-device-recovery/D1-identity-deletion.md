# D1 Eliminación de identidad

[Volver al índice](../README.md) · D · Identidad, dispositivo y recuperación

## Diagrama de estados

GitHub muestra directamente el bloque Mermaid de abajo; si no se ve bien, abra el SVG.

```mermaid
flowchart TD
  U(("El usuario confirma el borrado de la identidad")) --> P1["PlatformCredentialPending<br/>plan de borrado escrito, llamadas activas terminadas"]
  P1 -->|"[Usuario] revertir (solo en esta fase)"| RB["Borrado cancelado"]
  P1 -->|"[Plataforma] credenciales del llavero borradas"| P2["CoreFinalizePending"]
  P2 -->|"[Local] limpieza de Core terminada"| DONE["Borrado completo"]
  P2 -.- P2L1(["⟲ [Local] reanudar tras interrupción"])
  class P2L1 loop
  L(("Flujo de borrado heredado")) --> L1["LegacyCorePending"]
  L1 --> L2["LegacyPlatformSecretPending"]
  L2 -->|"[Usuario] confirmar el borrado del secreto heredado"| DONE
  class DONE,RB term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/D1.svg) · [Fuente Mermaid](diagrams/D1.mmd)

## Tabla de estados

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | El usuario confirma el borrado; Core escribe el plan de borrado; se terminan las llamadas activas de la identidad | La identidad sigue presente; el trabajo nuevo se bloquea | — | Revertir (solo en esta fase) | No | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | La plataforma confirma que se borraron las credenciales del llavero | Credenciales borradas; datos de Core pendientes de limpieza | — | Continuar (repetible) | No | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Puntos intermedios del flujo de borrado heredado | Igual que arriba | — | Continuar; confirmar el borrado del secreto heredado | No | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Borrado completo | Limpieza de Core terminada; fila de borrado eliminada | La identidad ya no existe | — | — | Sí | — |

«Igual que arriba» se refiere a la fila anterior de la tabla. En la columna de interfaz, los valores `xxx.yyy` son claves de localización.
