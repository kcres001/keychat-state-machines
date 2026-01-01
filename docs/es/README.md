# Máquinas de estados de Keychat: diagramas y tablas de estados

[English](../en/README.md) · **Español** · [Français](../fr/README.md) · [日本語](../ja/README.md) · [简体中文](../zh-Hans/README.md) · [繁體中文](../zh-Hant/README.md)

Este documento describe los flujos de varios pasos de Keychat que pueden quedar a la espera o implican fondos: los estados de cada flujo, cómo se entra en cada uno, dónde están los fondos o los datos, qué muestra la interfaz, qué puede hacer el usuario, qué estados son finales y quién provoca cada transición.

Base de código: Keychat iOS 1.5, commit `97c69dee1`. Todos los números de línea se refieren a ese commit.

## Cómo leerlo

### Leyenda

- **Verde**: Estado final
- **Amarillo**: Espera al usuario, o recuperable
- **Rombo azul violeta**: Comprobación previa / decisión
- **Recuadro redondeado discontinuo ⟲**: Bucle que permanece en el mismo estado (reintento, espera, pausa); la etiqueta indica el desencadenante
- **Flecha continua**: Transición; el prefijo entre corchetes indica el desencadenante (usuario, servidor, cadena / fuente de cadena, otro participante, mint, relay, notify, mantenimiento, recuperación al inicio, local)
- **Flecha discontinua**: Transición que solo ocurre bajo una condición concreta

«Igual que arriba» se refiere a la fila anterior de la tabla. En la columna de interfaz, los valores `xxx.yyy` son claves de localización.

### Abreviaturas de rutas

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

Un `:línea` suelto en una celda se refiere al archivo anterior de la misma celda.

## Índice

### A · Fondos on-chain y Ark

- [A1 Depósito en Ark, envío directo de BTC, aceleración RBF](ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 Envío de BTC desde Ark a una dirección on-chain](ark-onchain/A2-ark-to-onchain-send.md)
- [A3 Reserva de comisiones de salida](ark-onchain/A3-exit-fee-reservation.md)
- [A4 Salida forzada y reclamación](ark-onchain/A4-force-exit-and-claim.md)
- [A5 Actualización (refresh) y mantenimiento automático](ark-onchain/A5-refresh-and-maintenance.md)

### B · Pagos y cobros

- [B1 Envío y recepción Lightning (Ark)](payments/B1-lightning-ark.md)
- [B2 Recepción Cashu](payments/B2-cashu-receive.md)
- [B3 Transferencias en chat, sobres rojos y recibos](payments/B3-chat-transfer-red-packet.md)
- [B4 Franqueo de relay y recuperación](payments/B4-relay-postage-reclaim.md)
- [B5 Billetes de llamada de pago](payments/B5-paid-call-tickets.md)

### C · Mensajería y relaciones

- [C1 Envío de mensajes](messaging/C1-message-send.md)
- [C2 Subida y descarga de adjuntos](messaging/C2-attachments.md)
- [C3 Solicitudes de amistad y actualización PQXDH](messaging/C3-friend-requests.md)
- [C4 Invitaciones de grupo y activación del Welcome de MLS](messaging/C4-group-invites-mls.md)

### D · Identidad, dispositivo y recuperación

- [D1 Eliminación de identidad](identity-device-recovery/D1-identity-deletion.md)
- [D2 Activación del dispositivo, arranque y restauración de copia](identity-device-recovery/D2-device-activation-restore.md)
- [D3 Recuperación de la cartera](identity-device-recovery/D3-wallet-recovery.md)

- [Todos los estados (una sola tabla)](all-states.md)
