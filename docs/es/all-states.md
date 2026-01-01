# Todos los estados (una sola tabla)

[Volver al índice](README.md)

## A · Fondos on-chain y Ark

### [A1 Depósito en Ark, envío directo de BTC, aceleración RBF](ark-onchain/A1-onboard-direct-send-rbf.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | Se persiste en cuanto se genera la cotización; válida 300 s | PSBT sin firmar; los fondos siguen en la cartera on-chain, sin bloquear | `wallet.bitcoin.review` | Confirmar; o descartar la cotización (las cotizaciones de aceleración se pueden borrar) | No (sin limpieza al caducar; confirmar después de caducar se rechaza) | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | Firmada; `signed_tx_hex` escrito | Tx firmada guardada localmente; el envío directo ya se hizo `register_tx` en BDK y cuenta como gastado; entradas de onboard bloqueadas | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | Solo esperar | No | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | La fuente de cadena informa mempool | Tx en la mempool | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | Esperar; el envío directo admite RBF | No | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | Solo onboard: tx de financiación confirmada, Ark aún no la ha registrado | Fondos en la salida del board; VTXO del board bloqueado por el movement | Igual que arriba + `wallet.bitcoin.waiting-ark` | Esperar | No | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | Falló la consulta a la fuente de cadena, tx no visible, onboard sin cofirma o movement ausente | Envío directo: tx firmada aún en BDK; onboard: entradas bloqueadas | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | Esperar | No | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | Envío directo: un intento confirmado; onboard: movement correcto con prueba de salida | Envío directo: entregado; onboard: en Ark | `wallet.bitcoin.completed` | — | Sí (el envío directo puede volver a Unknown tras una reorganización de la cadena) | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | Solo onboard: cambian las condiciones del servidor (`submit_signed_board` devuelve None), o el movement de Bark queda Failed/Canceled | Si cambian las condiciones: las entradas siguen bloqueadas; si falla el movement: los fondos pueden estar en una salida | `wallet.bitcoin.failed` ("Transfer failed. Review a new quote to try again.") | Nueva cotización | Sí | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | Cofirma obtenida, VTXO construido | Tx de financiación pendiente de publicar; VTXO guardado como Locked | — | Visto en cadena → `Confirming`; entradas en conflicto con una tx confirmada → fallo; si no, en espera | No | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | Tx de financiación en mempool o en bloque | VTXO del board Locked, contabilizado en `pending_board` | — | Registrar cuando haya confirmaciones suficientes; reintentar si falla el registro; iniciar una salida cerca del vencimiento | No | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed` (tras la salida) | VTXO salido, o entradas gastadas por una tx en conflicto | Lo primero: en una salida; lo segundo: los fondos nunca salieron | — | Final | Sí | `BK/actions/board.rs:299-314` |
| A1-11 | Cotización de aceleración `Reviewing` + `replacement` | El usuario pide acelerar; se cumplen las condiciones | PSBT de reemplazo sin firmar, válida 300 s | — | Confirmar; o cancelar (solo borra el registro sin firmar) | No (sigue al linaje) | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | Padre con `replaced_by`, hijo firmado | El usuario confirma | Padre e hijo se confirman en una sola transacción | — | Esperar | No (sigue al linaje) | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | Estado del linaje | El intento que confirma pasa a Completed, los demás no cambian; la cabeza se juzga por mempool / no visto | — | — | — | No (sigue al linaje) | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 Envío de BTC desde Ark a una dirección on-chain](ark-onchain/A2-ark-to-onchain-send.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing` (Bark `Start` / `SplitWithArkoor`) | El usuario confirma; se escribe el checkpoint de la acción | VTXO de entrada bloqueados por la acción | `wallet.bitcoin-send.preparing` ("Payment in progress") | «Comprobar estado» | No | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing` (Bark `ArkoorRegistrationRequired`) | División arkoor hecha; entradas originales marcadas como gastadas | El VTXO de envío dividido **y el cambio** quedan bloqueados por la acción, a la espera de registro en el servidor | Igual que arriba | Igual que arriba | No | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing` (Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`) | Registrado; tx de offboard y forfeits intercambiados con el servidor | VTXO de envío bloqueado; cambio desbloqueado | Igual que arriba | Igual que arriba | No | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast` (Bark `AwaitingConfirmations`) | Tx de offboard publicada | Fondos en una tx on-chain; VTXO cedidos (forfeit) | `wallet.bitcoin-send.broadcast` | «Comprobar estado» | No | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | Movement correcto | Entregado en la dirección del destinatario | `wallet.bitcoin-send.confirmed` | — | Sí | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | Movement Failed / Canceled (Bark libera las entradas si falla en `Start`, `SplitWithArkoor` o un `ReadyForOffboard` previo al forfeit) | Entradas gastables de nuevo | `wallet.bitcoin-send.failed` | Nueva cotización | Sí | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | Resultado desconocido (la llamada de confirmación expiró o falló) | `start_send_onchain_confirmed` escribió el checkpoint pero expiró | Lo que contenga el estado de Bark subyacente | `wallet.bitcoin-send.uncertain` | Consultar la operación original; no se puede iniciar otra | No | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 Reserva de comisiones de salida](ark-onchain/A3-exit-fee-reservation.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`, `remaining_sats = Some(0)` | Cuenta nueva registrada; sin salidas por reclamar | Sin reserva | Ninguno | — | No (puede volver a Active) | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`, `Some(n)` | Hay una salida en curso y se pueden estimar las comisiones | n sat del BTC compartido se tratan como reserva y se restan del saldo disponible | La estructura de saldo tiene `reserved_sats` / `available_sats` (`W/ark/types.rs:429-439`) | Los envíos que invaden la reserva requieren confirmación explícita (`BitcoinExitReserveImpact`) | No | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`, `Some(último valor)` | Falló la estimación; se mantiene el último valor fiable | Igual que arriba | Igual que arriba | Igual que arriba | No | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`, `None` | Recuperación iniciada, nueva salida recuperada, primera salida detectada desde Idle, o nueva salida autorizada sin reestimación | Reserva desconocida; los envíos directos y los onboards en la misma red no están disponibles | Los envíos dan error | Ninguno | No | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | Salidas recuperadas pendientes de registro (`pending_recovered_exit_ids` no vacío) | Salidas recuperadas registradas, checkpoints de Bark incompletos | Igual que arriba | Igual que arriba | Ninguno | No | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 Salida forzada y reclamación](ark-onchain/A4-force-exit-and-claim.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | El usuario confirma la salida forzada, o el sistema sale automáticamente por un board / HTLC | Las tx de la cadena de salida se publican nivel a nivel; el BTC compartido paga el CPFP | `wallet.ark.exit-in-progress`, `wallet.ark.pending-funding`, `wallet.ark.refresh-to-progress-exit` | Actualizar para avanzar; añadir fondos on-chain | No | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | Todas las tx de salida confirmadas | Salida de salida, timelock aún no vencido | `wallet.ark.claim-timelock-running` | Esperar | No | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | Timelock vencido | Salida de salida, gastable solo por el usuario | `wallet.ark.claim-to-bitcoin-address` | Reclamar | No | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | Aparece en mempool una tx que gasta la salida | Tx de reclamación en la mempool | `wallet.ark.claim-in-progress` | Esperar | No; vuelve a `Claimable` si sale de la mempool | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | Tx de gasto confirmada | En la dirección del usuario | — | — | Sí | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO ya gastado off-chain; o el servidor barrió a mitad de la salida | Ninguno | — | — | Sí | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | Cancelada antes de publicar la tx final | VTXO aún gastable | — | Empezar de nuevo | Reanudable | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | El usuario confirma la cotización; tx firmada persistida; entradas registradas en `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | «Enviar la tx original» | No | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | Publicación iniciada (el estado se escribe antes de publicar) | — | Reclamando | Esperar | No | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | La publicación tuvo éxito, o la fuente de cadena informa mempool | — | `wallet.ark.claim-in-progress-message` | «Actualizar» | No | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired` (proyección) | Confirmada en cadena pero falló la escritura en la BD | — | `wallet.ark.claim-confirmation-checkpoint-required` | Actualizar | No | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | Confirmada en cadena y persistida | — | Reclamado | — | Sí | `claim_store.rs:287-293` |

### [A5 Actualización (refresh) y mantenimiento automático](ark-onchain/A5-refresh-and-maintenance.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | Actualización programada aceptada por el servidor, esperando la altura objetivo; espera programada: sí (`is_scheduled_waiting`) | Entradas no bloqueadas; aún admite salida forzada | — | — | No | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | Actualización delegada enviada, esperando a que el servidor ejecute una ronda; espera programada: no | Entradas bloqueadas por el movement, contabilizadas en `pending_in_round` | — | — | No | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | Ronda interactiva en cola / en curso; espera programada: no | Igual que arriba | — | — | No | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished` (no terminado) | Interacción terminada, esperando la confirmación de la tx de financiación de la ronda; espera programada: no | Igual que arriba; forfeits aún incompletos | — | — | No | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished` (terminado) → `Confirmed` | Tx de financiación confirmada y forfeits completos; espera programada: — | VTXO nuevos | — | — | Sí | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | Ronda fallida, cancelada por el usuario, o entradas perdidas sin poder reducirse; espera programada: — | Entradas liberadas | — | — | Sí | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | No hace falta actualizar / actualización automática desactivada | — | `disabled-title` | Activar, o actualizar manualmente | No (en reposo) | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | Plan aprobado / enviado | — | — | Esperar | No | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | Espera tras fallo 60 s → 30 min | — | — | Esperar | No | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | Resultado del envío desconocido; primero recuperar | — | — | Esperar (se revisa cada `URGENT_CHECK_SECONDS`) | No | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | Requiere autorización de comisión, revisión manual o más fondos | — | `authorization-*`, `funds-title`, etc. | Autorizar, revisar o añadir fondos | No (espera al usuario) | `W/ark/maintenance.rs:35-62` |

## B · Pagos y cobros

### [B1 Envío y recepción Lightning (Ark)](payments/B1-lightning-ark.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | El usuario confirma el pago | VTXO de entrada bloqueados por la acción | `InProgress` | Esperar | No | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | El servidor cofirmó las salidas HTLC | VTXO HTLC bloqueados, contabilizados en `pending_lightning_send` | `InProgress` | Esperar | No | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | Se ha pedido al servidor que pague | Igual que arriba | `InProgress` | Esperar | No | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | El servidor informa fallo, o sigue Pending cerca del vencimiento del HTLC | Igual que arriba, revocación preparada | `InProgress` | Esperar | No | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | El servidor se niega a revocar | Igual que arriba | `InProgress{has_failed_revocation: true}` | Esperar | No | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | Preimagen obtenida | Pagado | `Paid` | — | Sí | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | Revocación hecha (Done) | El servidor cofirma la revocación | De vuelta a VTXO gastables | — | — | Sí | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | Factura creada | Ninguno | — | — | No | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | El servidor preparó los VTXO HTLC-recv | Bloqueado; aún se puede abandonar | — | — | No; se abandona si se cierra la ventana de reclamación o se rechaza | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | Preimagen revelada al servidor | VTXO HTLC-recv bloqueados, contabilizados en `claimable_lightning_receive` | — | — | No; queda en espera tras una reclamación fallida | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | Las salidas reclamadas deben reenviarse a un tercero | Bloqueado | — | — | No | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | Reclamación completa | VTXO nuevos | — | — | Sí | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Recepción Cashu](payments/B2-cashu-receive.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | Token recibido, mint registrado | Token al portador en el almacén local cifrado (la única copia reutilizable) | Recepción en curso | Esperar | No | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked` (`UnknownMint`) | Mint no registrado | Igual que arriba | Hace falta añadir el mint | Añadir el mint y continuar | No | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | Conciliando, o el SDK ya tiene una saga para ello | Token + saga de CDK | Procesando | Esperar | No | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue` (`RetryRequired`) | La conciliación falló sin id de operación del SDK; reintento en 30 s | Token | Procesando | Esperar | No | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal` (sin incidencia) | Se obtuvo `Credited` / `AlreadyReceived` / `Recovered` | En el saldo de CDK | Recibido | — | Sí | `receive_operations.rs:196-206` |
| B2-06 | `Terminal` (`AlreadySpent`) | El mint dice que el token ya se gastó | Ninguno | Error `CashuTokenAlreadySpent` | — | Sí | `receive_operations.rs:207-213` |

### [B3 Transferencias en chat, sobres rojos y recibos](payments/B3-chat-transfer-red-packet.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`, sin operación del proveedor | El usuario confirma el importe | No se han movido fondos | — | Continuar | No | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`, con operación del proveedor (Cashu) | Saga de envío Cashu iniciada | Saga en el mint; el token puede existir ya | `wallet.chat.previous-cashu-interrupted` | «Continuar»: recuperar el token de la saga | No | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`, resultado de Ark sin resolver | Resultado de la transferencia Ark desconocido | Puede haber salido ya de la cuenta | `wallet.chat.previous-wallet-transfer-interrupted` | Conciliar por id de tx; o abandonar tras confirmar que no se envió nada | No | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | Token (Cashu) o id de tx (Ark) obtenido | Los fondos salieron de la cartera; token en la saga local | `wallet.chat.wallet-transfer-complete` (se puede reanudar el traspaso del mensaje) | Seguir enviando el mensaje | No | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | Borrado (hecho) | Mensaje entregado al emisor | Registro de traspaso borrado | — | — | Sí | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | ninguno → `Pending` / `Available` | Recibido o enviado | — | Reclamable | El destinatario reclama; el remitente no puede reclamar (sobre rojo) | No | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | El destinatario pulsa reclamar; la fase se guarda antes de canjear | — | Reclamando, `can_retry` | Reintentar | No | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | Canjeado y recibo enviado; o recibo del otro participante recibido | — | Reclamado | Reenviar el recibo si falló | Sí | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | El remitente inicia el reembolso tras caducar el sobre rojo | — | `chat.payment.shared.refunding` / `continue-refund` | Continuar el reembolso (la interfaz lo reanuda sola) | No | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | Reembolso completo | — | Reembolsado | Reenviar el recibo si falló | Sí | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed` (observado por el remitente) | El remitente ve el token gastado sin recibo | — | `payment_spent_without_receipt` | — | Sí (proyección) | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 Franqueo de relay y recuperación](payments/B4-relay-postage-reclaim.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | El relay no cobra franqueo | — | — | — | Sí (para esa entrega) | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | Aún sin pagar, o no se puede pagar | No pagado | — | — | No | same as above |
| B4-03 | `Created` | Token de franqueo creado y enviado al relay con el evento | Token entregado al relay; el libro de la cartera registra un envío `relay_stamp` | — | — | Sí (para esa entrega) | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | No recuperable (sin vínculo) | El estado de la fila de entrega no es failed / timeout / no_ack / terminal_failed / retry_exhausted, o se recibió un OK | — | No se muestra la opción | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | El vínculo corresponde exactamente a una saga de envío `TokenCreated` | Puede que el relay aún no haya canjeado el token | Se muestra «Recuperar» | Recuperar | No | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | Intención de recuperación aceptada (persistida antes de tocar el mint), o estado del mint incierto / inalcanzable | Intención + pruebas originales | Procesando | Recuperar de nuevo | No | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | Intercambio completo | De vuelta en la cartera | Recuperado | — | Sí | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | Todas las pruebas gastadas (el relay las canjeó), o sin entrada única en el libro | En el relay | No recuperable | — | Sí | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 Billetes de llamada de pago](payments/B5-paid-call-tickets.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | El usuario confirma la compra; fila del pedido escrita | Token aún no emitido; o emitido pero aún no escrito | Pagando | Continuar / cancelar | No | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | Token escrito en el pedido y entregado a notify | Token en el pedido | Confirmando; la página de confirmación solo ofrece Continue | Seguir consultando | No | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify confirmó; después se proyecta según el uso | Pase | Disponible / en uso / usado | Llamar | Sí (lado del pago) | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify devolvió fallo, o la cartera no pudo emitir el token | Token borrado | Fallido | — | Sí | `rtc_tickets.rs:118-125`, `:152` |

## C · Mensajería y relaciones

### [C1 Envío de mensajes](messaging/C1-message-send.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | Intención de vista previa de enlace sin terminar | Trabajo de envío | Enviando | Texto / archivo se pueden cancelar | No | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | Intención de adjunto sin terminar | Trabajo de envío + adjuntos locales | Subiendo | Igual que arriba | No | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | Existe el trabajo, aún no entregado al transporte | Trabajo de envío | Enviando | Igual que arriba | No | `delivery_queries.rs:430-432` |
| C1-04 | `Failed` (dentro del trabajo) | Subida en pausa | Trabajo de envío | Fallido | Reintentar | No | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | Eventos de transporte creados; ni éxito de negocio ni final | `app_message_transport_events` + filas de entrega por relay | Enviando | Esperar | No | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | La unidad principal alcanzó el éxito de negocio | Filas de entrega | Enviado | — | Sí | `delivery_queries.rs:433-437` |
| C1-07 | `Failed` (entrega) | La política de transporte lo declara final | Filas de entrega | Fallido | Reintento manual (según `manual_retryable`) | Sí | `delivery_queries.rs:438-442` |

### [C2 Subida y descarga de adjuntos](messaging/C2-attachments.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | No descargado | URL remota + clave | Descargable | Descarga | No | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | Descarga iniciada (`transfer_state=1`) | Archivo temporal | Descargando | Esperar | No | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` y el archivo local existe | Archivo local | Se puede abrir | — | Sí (vuelve a la proyección Remote si falta el archivo) | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | La descarga falló, o se detectó una interrupción al arrancar / reiniciar el runtime | `last_error_code` | Fallido | Reintentar | No | `attachments.rs:462-469`, `:472-492` |
| C2-05 | Subiendo | Intención de adjunto sin terminar | — | `Uploading` | Los mensajes de texto / archivo se pueden cancelar | No | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | En pausa (fallo dentro del trabajo) | El fallo de subida escribe `upload_state='paused'` | — | `Failed` | Reintento permitido para errores reintentables; no para origen ausente, intención inválida o destino inseguro | No | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 Solicitudes de amistad y actualización PQXDH](messaging/C3-friend-requests.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending` (lado del remitente) | Solicitud enviada al planificador de reintentos de entrega | Solicitud pendiente + sesión pendiente a nivel de protocolo | Esperando al otro participante | Retirar | No | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | Al menos un relay la aceptó | Igual que arriba | Esperando al otro participante | Retirar | No | `A/types.rs:3512` |
| C3-03 | Aceptada por el otro participante | Respuesta del otro participante recibida | Sesión establecida | Contacto | — | Sí | — |
| C3-04 | Retirada | El usuario la retira; o una nueva solicitud al mismo participante la sustituye | Sesión pendiente de protocolo y transporte borrados | — | — | Sí | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | Reintento de actualización PQXDH | El usuario lo lanza manualmente | — | Avisos distintos para `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` | Reintentar más tarde | Devuelve en cada llamada | `A/types.rs:533-540` |

### [C4 Invitaciones de grupo y activación del Welcome de MLS](messaging/C4-group-invites-mls.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | Invitación enviada o recibida; el reintento de entrega vuelve a pending | — | — | Aceptar / rechazar / retirar | No | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | Falló la entrega, o falló el procesamiento de la decisión | — | — | El remitente MLS puede reintentar la entrega (`group_invite_lifecycle_allows_delivery_retry`) | No | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | Aceptada; esperando a que llegue el mensaje de control de activación del miembro | — | `PendingAcknowledgement` | Esperar | No | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | Se recibió evidencia de activación autenticada o el OK del relay | — | `Active` | — | Sí | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | Rechazada, retirada, caducada | — | — | — | Sí | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | Pendiente de confirmación durante el borrado | — | — | — | No | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · Identidad, dispositivo y recuperación

### [D1 Eliminación de identidad](identity-device-recovery/D1-identity-deletion.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | El usuario confirma el borrado; Core escribe el plan de borrado; se terminan las llamadas activas de la identidad | La identidad sigue presente; el trabajo nuevo se bloquea | — | Revertir (solo en esta fase) | No | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | La plataforma confirma que se borraron las credenciales del llavero | Credenciales borradas; datos de Core pendientes de limpieza | — | Continuar (repetible) | No | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Puntos intermedios del flujo de borrado heredado | Igual que arriba | — | Continuar; confirmar el borrado del secreto heredado | No | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Borrado completo | Limpieza de Core terminada; fila de borrado eliminada | La identidad ya no existe | — | — | Sí | — |

### [D2 Activación del dispositivo, arranque y restauración de copia](identity-device-recovery/D2-device-activation-restore.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | Inicio | — | — | Esperar | No | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | Sin datos | — | — | Crear o restaurar | No (espera al usuario) | `:63` |
| D2-03 | `ActivationRequired{pending}` | Hay datos, activación sin verificar | — | — | Activar | No (espera al usuario) | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | Restauración bloqueada, falta la clave de la BD, almacenamiento no disponible / incompatible, llavero no disponible, credenciales en conflicto, política no soportada | — | — | Actuar según el motivo | No (espera al usuario) | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | Todo listo; o en pausa manual / como origen de migración / como destino de migración | — | — | — | Sí (en reposo) | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging` (iOS: `preparing` / `prepared`) | — | Área de preparación; datos originales intactos | — | Área de preparación limpiada al arrancar | No | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain` (iOS: `replacement_started` / `restored_data_installed`) | — | Instantánea de los datos originales + datos nuevos | — | Se revierte a la instantánea al arrancar; luego el diario se marca committed para no repetir la reversión | No | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking` (iOS: `validated`) | — | Datos nuevos | — | Igual que arriba (reversión) | No | same as above |
| D2-09 | `RestoreComplete` (iOS: `committed`) | — | Datos nuevos; instantánea limpiada | — | — | Sí | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed` (iOS: —) | — | Depende del punto de fallo | — | Solo puede volver a Idle | Sí | `A/backup_restore.rs:698-741` |

### [D3 Recuperación de la cartera](identity-device-recovery/D3-wallet-recovery.md)

| ID | Estado (nombre en código) | Condición de entrada | Dónde están los fondos / datos | Interfaz | Qué puede hacer el usuario | ¿Final? | Ubicación en el código |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | Todas las épocas completas | VTXO y salidas recuperados | — | — | Sí | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | El servidor no tiene registro de la cartera | — | — | Probar otra fuente | Sí | `types.rs:70` |
| D3-03 | `Incomplete` | Alguna época incompleta; la cuenta sigue `recovery_pending`, operaciones cooperativas rechazadas, reserva None | Recuperada en parte | — | Recuperar de nuevo | No | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | Servidor no disponible / la identidad del servidor cambió | — | — | Reintentar más tarde / juzgar manualmente | Sí (este intento) | `types.rs:72-73` |
| D3-05 | Escaneo de claves `Ready` / `LimitReached` / `CandidatesMatched` | Escaneado hasta el límite o coincidencia por índice | Progreso del escaneo | — | Ampliar el escaneo | No | `types.rs:89-108` |
| D3-06 | Escaneo del historial BTC `Ready` / `LimitReached` / `UnsupportedChainSource` | Igual que arriba (BTC compartido) | Direcciones descubiertas | — | Ampliar el escaneo | No | `types.rs:441-457` |
