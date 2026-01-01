# Tous les états (tableau unique)

[Retour au sommaire](README.md)

## A · Fonds on-chain et Ark

### [A1 Dépôt vers Ark, envoi direct de BTC, accélération RBF](ark-onchain/A1-onboard-direct-send-rbf.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| A1-01 | `Reviewing` | Persisté dès la génération du devis ; valable 300 s | PSBT non signé ; les fonds restent dans le portefeuille on-chain, non verrouillés | `wallet.bitcoin.review` | Confirmer ; ou abandonner le devis (les devis d'accélération peuvent être supprimés) | Non (pas de nettoyage à l'expiration ; une confirmation après expiration est refusée) | `W/ark/bitcoin/transfer.rs:308-336`, `:355-358` |
| A1-02 | `Submitting` | Signé ; `signed_tx_hex` écrit | Tx signée stockée localement ; l'envoi direct est déjà `register_tx` dans BDK et compté comme dépensé ; entrées de l'onboard verrouillées | `wallet.bitcoin.submitting` + `wallet.bitcoin.unknown-help` | Attendre uniquement | Non | direct send `W/ark/bitcoin/transfer.rs:382-393`; onboard `W/ark/bark_runtime/onboard.rs:220-236` |
| A1-03 | `Broadcast` | La source de chaîne signale le mempool | Tx dans le mempool | `wallet.bitcoin.onboard.awaiting-confirmation` + `wallet.bitcoin.waiting-chain` | Attendre ; l'envoi direct peut faire un RBF | Non | direct `W/ark/bitcoin/replacement.rs:803-807`; onboard `W/ark/bark_runtime/onboard.rs:339` |
| A1-04 | `WaitingForArk` | Onboard uniquement : tx de financement confirmée, Ark ne l'a pas encore enregistrée | Fonds dans la sortie du board ; VTXO du board verrouillé par le movement | Idem + `wallet.bitcoin.waiting-ark` | Attendre | Non | `W/ark/bark_runtime/onboard.rs:336-338` |
| A1-05 | `Unknown` | Échec de la requête à la source de chaîne, tx invisible, onboard sans cosignature, ou movement absent | Envoi direct : tx signée toujours dans BDK ; onboard : entrées verrouillées | `wallet.bitcoin.checking` + `wallet.bitcoin.unknown-help` | Attendre | Non | direct `W/ark/bitcoin/replacement.rs:806`, `:808-809`; onboard `W/ark/bark_runtime/onboard.rs:311-313`, `:340-345`; projected as Unknown without chain source `W/ark/provider/shared_bitcoin.rs:818-840` |
| A1-06 | `Completed` | Envoi direct : une tentative confirmée ; onboard : movement réussi avec preuve de sortie | Envoi direct : livré ; onboard : dans Ark | `wallet.bitcoin.completed` | — | Oui (l'envoi direct peut revenir à Unknown après une réorganisation de la chaîne) | `W/ark/bitcoin/replacement.rs:796-802`; `W/ark/bark_runtime/onboard.rs:325-327` |
| A1-07 | `Failed` | Onboard uniquement : conditions du serveur modifiées (`submit_signed_board` renvoie None), ou movement Bark Failed/Canceled | Si les conditions changent : les entrées restent verrouillées ; si le movement échoue : les fonds peuvent être dans une sortie | `wallet.bitcoin.failed` ("Transfer failed. Review a new quote to try again.") | Nouveau devis | Oui | `W/ark/bark_runtime/onboard.rs:306-310`, `:328-329` |
| A1-08 | `Broadcasting` | Cosignature obtenue, VTXO construit | Tx de financement à diffuser ; VTXO stocké comme Locked | — | Vue sur la chaîne → `Confirming` ; entrées en conflit avec une tx confirmée → échec ; sinon en attente | Non | `BK/actions/board.rs:147-160`, `:230-283` |
| A1-09 | `Confirming` | Tx de financement dans le mempool ou un bloc | VTXO du board Locked, compté dans `pending_board` | — | Enregistrer dès que les confirmations suffisent ; réessayer en cas d'échec d'enregistrement ; lancer une sortie à l'approche de l'expiration | Non | `BK/actions/board.rs:290-393` |
| A1-10 | movement `Failed` (après la sortie) | VTXO sorti, ou entrées dépensées par une tx en conflit | Premier cas : dans une sortie ; second cas : les fonds ne sont jamais partis | — | Terminal | Oui | `BK/actions/board.rs:299-314` |
| A1-11 | Devis d'accélération `Reviewing` + `replacement` | L'utilisateur demande une accélération ; conditions remplies | PSBT de remplacement non signé, valable 300 s | — | Confirmer ; ou annuler (supprime seulement l'enregistrement non signé) | Non (suit la lignée) | `W/ark/bitcoin/replacement.rs:378-401`, `:403-450` |
| A1-12 | Parent avec `replaced_by`, enfant signé | L'utilisateur confirme | Parent et enfant validés dans une seule transaction | — | Attendre | Non (suit la lignée) | `W/ark/bitcoin/replacement.rs:825-860` |
| A1-13 | État de la lignée | La tentative confirmée passe à Completed, les autres restent inchangées ; la tête est jugée selon mempool / non vue | — | — | — | Non (suit la lignée) | `W/ark/bitcoin/replacement.rs:786-823` |

### [A2 Envoi de BTC depuis Ark vers une adresse on-chain](ark-onchain/A2-ark-to-onchain-send.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| A2-01 | `Preparing` (Bark `Start` / `SplitWithArkoor`) | L'utilisateur confirme ; le point de contrôle de l'action est écrit | VTXO d'entrée verrouillés par l'action | `wallet.bitcoin-send.preparing` ("Payment in progress") | « Vérifier l'état » | Non | projection `W/ark/bark_runtime/bitcoin_send.rs:105-111`, `:127-139`; locking `BK/actions/offboard.rs:553-572` |
| A2-02 | `Preparing` (Bark `ArkoorRegistrationRequired`) | Découpage arkoor effectué ; entrées d'origine marquées dépensées | Le VTXO d'envoi découpé **et la monnaie** sont tous deux verrouillés par l'action, en attente d'enregistrement auprès du serveur | Idem | Idem | Non | `BK/actions/offboard.rs:605-636`, `:641-661` |
| A2-03 | `Preparing` (Bark `ReadyForOffboard` / `OffboardTxPrepared` / `ReadyForBroadcast`) | Enregistré ; tx d'offboard et forfeits échangés avec le serveur | VTXO d'envoi verrouillé ; monnaie déverrouillée | Idem | Idem | Non | `BK/actions/offboard.rs:263-286`, `:670-760` |
| A2-04 | `Broadcast` (Bark `AwaitingConfirmations`) | Tx d'offboard diffusée | Fonds dans une tx on-chain ; VTXO cédés (forfeit) | `wallet.bitcoin-send.broadcast` | « Vérifier l'état » | Non | `bitcoin_send.rs:109`; `BK/actions/offboard.rs:287-326` |
| A2-05 | `Confirmed` | Movement réussi | Livré à l'adresse du destinataire | `wallet.bitcoin-send.confirmed` | — | Oui | `bitcoin_send.rs:106` |
| A2-06 | `Failed` | Movement Failed / Canceled (Bark libère les entrées en cas d'échec dans `Start`, `SplitWithArkoor` ou un `ReadyForOffboard` antérieur au forfeit) | Entrées à nouveau dépensables | `wallet.bitcoin-send.failed` | Nouveau devis | Oui | `bitcoin_send.rs:107`; `BK/actions/offboard.rs:352-414` |
| A2-07 | Résultat inconnu (l'appel de confirmation a expiré ou échoué) | `start_send_onchain_confirmed` a écrit le point de contrôle mais a expiré | Selon l'état Bark sous-jacent | `wallet.bitcoin-send.uncertain` | Vérifier l'opération d'origine ; impossible d'en lancer une autre | Non | `bitcoin_send.rs:73-79`; `iOS/Wallet/Views/Forms/ArkBitcoinSendReview.swift:240-242` |

### [A3 Réserve de frais de sortie](ark-onchain/A3-exit-fee-reservation.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| A3-01 | `Idle`, `remaining_sats = Some(0)` | Nouveau compte enregistré ; aucune sortie à réclamer | Aucune réserve | Aucun | — | Non (peut redevenir Active) | registration `W/ark/bitcoin.rs:259-271`; zeroing `W/ark/bark_runtime/exit_fee.rs:195-200` |
| A3-02 | `Active`, `Some(n)` | Une sortie est en cours et les frais peuvent être estimés | n sat du BTC partagé sont réservés et déduits du solde disponible | La structure de solde comporte `reserved_sats` / `available_sats` (`W/ark/types.rs:429-439`) | Les envois qui empiètent sur la réserve demandent une confirmation explicite (`BitcoinExitReserveImpact`) | Non | `exit_fee.rs:215-221`, `:116-122` |
| A3-03 | `Unavailable`, `Some(dernière valeur)` | Estimation échouée ; dernière valeur fiable conservée | Idem | Idem | Idem | Non | `exit_fee.rs:207-212`; retention rule `W/ark/bitcoin.rs:322-330` |
| A3-04 | `Unavailable`, `None` | Récupération lancée, nouvelle sortie récupérée, première sortie détectée depuis Idle, ou nouvelle sortie autorisée sans réestimation | Réserve inconnue ; les envois directs et les onboards sur le même réseau sont indisponibles | Les envois échouent | Aucun | Non | `W/ark/bitcoin.rs:62-77`, `:245-251`, `:259-267`; `exit_fee.rs:123-131`; error site `W/ark/bitcoin/transfer.rs:232-235` |
| A3-05 | Sorties récupérées en attente d'enregistrement (`pending_recovered_exit_ids` non vide) | Sorties récupérées enregistrées, points de contrôle Bark incomplets | Idem | Idem | Aucun | Non | `W/ark/bitcoin.rs:69`; rejected on read `transfer.rs:231-234`; cleared `exit_fee.rs:92-105` |

### [A4 Sortie forcée et réclamation](ark-onchain/A4-force-exit-and-claim.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| A4-01 | `Start` / `Processing` | L'utilisateur confirme la sortie forcée, ou le système sort automatiquement pour un board / HTLC | Les tx de la chaîne de sortie sont diffusées niveau par niveau ; le BTC partagé paie le CPFP | `wallet.ark.exit-in-progress`, `wallet.ark.pending-funding`, `wallet.ark.refresh-to-progress-exit` | Rafraîchir pour avancer ; ajouter des fonds on-chain | Non | `BK/exit/progress/mod.rs:45-56`; funding check `W/ark/bark_runtime/exit_fee.rs:225-250` |
| A4-02 | `AwaitingDelta` | Toutes les tx de sortie confirmées | Sortie de sortie, timelock pas encore échu | `wallet.ark.claim-timelock-running` | Attendre | Non | `BK/exit/progress/mod.rs:57` |
| A4-03 | `Claimable` | Timelock échu | Sortie de sortie, dépensable uniquement par l'utilisateur | `wallet.ark.claim-to-bitcoin-address` | Réclamer | Non | `BK/exit/progress/states.rs:356-361` |
| A4-04 | `ClaimInProgress` | Une tx du mempool dépensant la sortie apparaît | Tx de réclamation dans le mempool | `wallet.ark.claim-in-progress` | Attendre | Non ; revient à `Claimable` si elle sort du mempool | `BK/exit/progress/states.rs:367-389` |
| A4-05 | `Claimed` | Tx de dépense confirmée | À l'adresse de l'utilisateur | — | — | Oui | `BK/exit/progress/states.rs:344-349`, `:375-378` |
| A4-06 | `VtxoAlreadySpent` / `VtxoSwept` | VTXO déjà dépensé hors chaîne ; ou balayé par le serveur en cours de sortie | Aucun | — | — | Oui | `BK/exit/models/mod.rs:40-47` |
| A4-07 | `Canceled` | Annulée avant la diffusion de la tx finale | VTXO toujours dépensable | — | Recommencer | Reprenable | `BK/exit/models/mod.rs:48-51` |
| A4-08 | `Prepared` | L'utilisateur confirme le devis ; tx signée persistée ; entrées enregistrées dans `ark_claim_inputs` | — | `wallet.ark.claim-prepared` | « Soumettre la tx d'origine » | Non | `W/ark/bark_runtime/exit_claim.rs:46-61`; `W/ark/claim_store.rs:189-218` |
| A4-09 | `Submitting` | Diffusion lancée (l'état est écrit avant la diffusion) | — | Réclamation en cours | Attendre | Non | `exit_claim.rs:97-104`; `claim_store.rs:262-271` |
| A4-10 | `Submitted` | Diffusion réussie, ou la source de chaîne signale le mempool | — | `wallet.ark.claim-in-progress-message` | « Actualiser » | Non | `claim_store.rs:272-279`; `exit_claim.rs:140-142` |
| A4-11 | `ConfirmationCheckpointRequired` (projection) | Confirmée sur la chaîne mais l'écriture en base a échoué | — | `wallet.ark.claim-confirmation-checkpoint-required` | Actualiser | Non | `exit_claim.rs:130-139`, `:153-159` |
| A4-12 | `Confirmed` | Confirmée sur la chaîne et persistée | — | Réclamé | — | Oui | `claim_store.rs:287-293` |

### [A5 Rafraîchissement et maintenance automatique](ark-onchain/A5-refresh-and-maintenance.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| A5-01 | `NonInteractivePending{scheduled_height: Some}` | Rafraîchissement planifié accepté par le serveur, en attente de la hauteur cible ; attente planifiée : oui (`is_scheduled_waiting`) | Entrées non verrouillées ; sortie forcée toujours possible | — | — | Non | `BK/round/mod.rs:262-267` |
| A5-02 | `NonInteractivePending{scheduled_height: None}` / `Redelegating` | Rafraîchissement délégué soumis, en attente d'un round du serveur ; attente planifiée : non | Entrées verrouillées par le movement, comptées dans `pending_in_round` | — | — | Non | `BK/round/mod.rs:1043-1059`, `:816-830`; balance `BK/balance.rs:190-197` |
| A5-03 | `InteractivePending` / `InteractiveOngoing` | Round interactif en file / en cours ; attente planifiée : non | Idem | — | — | Non | `BK/round/mod.rs:777`, `:783-803` |
| A5-04 | `Finished` (non terminé) | Interaction terminée, en attente de confirmation de la tx de financement du round ; attente planifiée : non | Idem ; forfeits pas encore terminés | — | — | Non | `BK/round/mod.rs:923-959` |
| A5-05 | `Finished` (terminé) → `Confirmed` | Tx de financement confirmée et forfeits terminés ; attente planifiée : — | Nouveaux VTXO | — | — | Oui | `BK/round/mod.rs:771-775`, `:937-948` |
| A5-06 | `Failed` / `Canceled` | Round échoué, annulé par l'utilisateur, ou entrées perdues sans réduction possible ; attente planifiée : — | Entrées libérées | — | — | Oui | `BK/round/mod.rs:793-813` |
| A5-07 | `Idle` / `Disabled` | Aucun rafraîchissement nécessaire / rafraîchissement automatique désactivé | — | `disabled-title` | Activer, ou rafraîchir manuellement | Non (au repos) | `W/ark/maintenance.rs:72-75` |
| A5-08 | `Ready` / `Refreshing` | Plan approuvé / soumis | — | — | Attendre | Non | `W/ark/provider/maintenance.rs:530-538` |
| A5-09 | `Retrying` | Attente après échec 60 s → 30 min | — | — | Attendre | Non | `W/ark/maintenance.rs:132-146` |
| A5-10 | `RecoveryRequired` | Résultat de la soumission inconnu ; récupérer d'abord | — | — | Attendre (revérifié toutes les `URGENT_CHECK_SECONDS`) | Non | `W/ark/provider/maintenance.rs:241-258` |
| A5-11 | `AuthorizationRequired` / `ManualReview` / `NeedsFunds` | Nécessite une autorisation de frais, une revue manuelle ou plus de fonds | — | `authorization-*`, `funds-title`, etc. | Autoriser, examiner ou ajouter des fonds | Non (en attente de l'utilisateur) | `W/ark/maintenance.rs:35-62` |

## B · Paiements et encaissements

### [B1 Envoi et réception Lightning (Ark)](payments/B1-lightning-ark.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| B1-01 | `Start` / `InputsLocked` | L'utilisateur confirme le paiement | VTXO d'entrée verrouillés par l'action | `InProgress` | Attendre | Non | `BK/actions/lightning/pay.rs:326-330`, `:221-229` |
| B1-02 | `HtlcReceived` | Le serveur a cosigné les sorties HTLC | VTXO HTLC verrouillés, comptés dans `pending_lightning_send` | `InProgress` | Attendre | Non | `BK/actions/lightning/pay.rs:331-332` |
| B1-03 | `PaymentInitiated` | Le serveur a reçu l'ordre de payer | Idem | `InProgress` | Attendre | Non | `BK/actions/lightning/pay.rs:183-207` |
| B1-04 | `RevocableHtlcs` | Le serveur signale un échec, ou reste Pending près de l'expiration du HTLC | Idem, révocation préparée | `InProgress` | Attendre | Non | `BK/actions/lightning/pay.rs:189-198` |
| B1-05 | `RevocationStuck` | Le serveur refuse de révoquer | Idem | `InProgress{has_failed_revocation: true}` | Attendre | Non | `BK/actions/lightning/pay.rs:260-275`, `:299-315`; `W/ark/bark_runtime.rs:3032-3038`; `iOS/Wallet/Ark/ArkWalletModels.swift:1117` |
| B1-06 | `Paid` | Préimage obtenue | Payé | `Paid` | — | Oui | `BK/actions/lightning/pay.rs:185-188` |
| B1-07 | Révocation effectuée (Done) | Le serveur cosigne la révocation | Retour à des VTXO dépensables | — | — | Oui | `BK/actions/lightning/pay.rs:209-213` |
| B1-08 | `AwaitingPayment` | Facture créée | Aucun | — | — | Non | `BK/actions/lightning/receive.rs:257-258` |
| B1-09 | `HtlcsReady` | Le serveur a préparé les VTXO HTLC-recv | Verrouillé ; encore abandonnable | — | — | Non ; abandonné si la fenêtre de réclamation se ferme ou si la réclamation est refusée | `BK/actions/lightning/receive.rs:259-262`, `:204-211`, `:240-246` |
| B1-10 | `PreimageRevealed` | Préimage révélée au serveur | VTXO HTLC-recv verrouillés, comptés dans `claimable_lightning_receive` | — | — | Non ; en attente après une réclamation échouée | `BK/actions/lightning/receive.rs:263-269`, `:212-224`, `:247-249`; `BK/lightning/receive.rs:205-240` |
| B1-11 | `Delivering` | Les sorties réclamées doivent être transmises à un tiers | Verrouillé | — | — | Non | `BK/actions/lightning/receive.rs:270-273` |
| B1-12 | `Settled` | Réclamation terminée | Nouveaux VTXO | — | — | Oui | `W/ark/bark_runtime.rs:3007-3014` |

### [B2 Réception Cashu](payments/B2-cashu-receive.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| B2-01 | `Accepted` | Jeton reçu, mint enregistré | Jeton au porteur dans le stockage local chiffré (la seule copie rejouable) | Réception en cours | Attendre | Non | `W/cashu/receive_store.rs:340-345`; `W/cashu/receive_operations.rs:63` |
| B2-02 | `Blocked` (`UnknownMint`) | Mint non enregistré | Idem | Ajout du mint requis | Ajouter le mint, puis continuer | Non | `receive_store.rs:342-345`; skipped by maintenance `receive_operations.rs:509-512` |
| B2-03 | `Reconciling` | Rapprochement en cours, ou le SDK a déjà une saga pour cela | Jeton + saga CDK | Traitement | Attendre | Non | `receive_operations.rs:188-194`, `:216-226` |
| B2-04 | `RetryDue` (`RetryRequired`) | Le rapprochement a échoué sans id d'opération SDK ; nouvel essai dans 30 s | Jeton | Traitement | Attendre | Non | `receive_operations.rs:192`, `:214-227` |
| B2-05 | `Terminal` (sans incident) | `Credited` / `AlreadyReceived` / `Recovered` obtenu | Dans le solde CDK | Reçu | — | Oui | `receive_operations.rs:196-206` |
| B2-06 | `Terminal` (`AlreadySpent`) | Le mint indique que le jeton est déjà dépensé | Aucun | Erreur `CashuTokenAlreadySpent` | — | Oui | `receive_operations.rs:207-213` |

### [B3 Virements dans le chat, enveloppes rouges et reçus](payments/B3-chat-transfer-red-packet.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| B3-01 | `Preparing`, sans opération du fournisseur | L'utilisateur confirme le montant | Aucun fonds déplacé | — | Continuer | Non | `W/ledger.rs:539-606` |
| B3-02 | `Preparing`, avec opération du fournisseur (Cashu) | Saga d'envoi Cashu lancée | Saga côté mint ; le jeton existe peut-être déjà | `wallet.chat.previous-cashu-interrupted` | « Continuer » : récupérer le jeton depuis la saga | Non | `WU/wallet.rs:2752-2810`, `:1880-1931` |
| B3-03 | `Preparing`, résultat Ark non résolu | Résultat du virement Ark inconnu | Peut avoir déjà quitté le compte | `wallet.chat.previous-wallet-transfer-interrupted` | Rapprocher par id de tx ; ou abandonner après avoir confirmé que rien n'a été envoyé | Non | `W/ledger.rs:734-759`; `iOS/Wallet/Views/ChatPayFlowSheet.swift:862-900` |
| B3-04 | `Prepared` | Jeton (Cashu) ou id de tx (Ark) obtenu | Les fonds ont quitté le portefeuille ; jeton dans la saga locale | `wallet.chat.wallet-transfer-complete` (le transfert du message peut reprendre) | Poursuivre l'envoi du message | Non | `W/ledger.rs:691-722`; `ChatPayFlowSheet.swift:725-731` |
| B3-05 | Supprimé (terminé) | Message remis à l'expéditeur | Enregistrement de transfert supprimé | — | — | Oui | `W/ledger.rs:724-732`; `ChatPayFlowSheet.swift:851` |
| B3-06 | aucun → `Pending` / `Available` | Reçu ou envoyé | — | Réclamable | Le destinataire réclame ; l'expéditeur ne peut pas réclamer (enveloppe rouge) | Non | `A/payment_lifecycle.rs:826-849` |
| B3-07 | `Accepting` → `Pending` / `Accepting` | Le destinataire touche « réclamer » ; la phase est enregistrée avant l'encaissement | — | Réclamation en cours, `can_retry` | Réessayer | Non | `A/messaging/payment_messages.rs:876-935`; projection `payment_lifecycle.rs:843-844`, `:858` |
| B3-08 | `Accepted` → `Succeeded` / `Accepted` | Encaissé et reçu envoyé ; ou reçu du correspondant reçu | — | Réclamé | Renvoyer le reçu en cas d'échec | Oui | `payment_lifecycle.rs:417-455`, `:828-841` |
| B3-09 | `Refunding` → `Expired` / `Accepting` | L'expéditeur lance le remboursement après expiration de l'enveloppe rouge | — | `chat.payment.shared.refunding` / `continue-refund` | Poursuivre le remboursement (l'interface reprend automatiquement) | Non | `payment_lifecycle.rs:470-480`; `iOS/Views/Components/CashuBubble.swift:723-726`, `:330`, `:1007-1025` |
| B3-10 | `Refunded` | Remboursement terminé | — | Remboursé | Renvoyer le reçu en cas d'échec | Oui | `payment_lifecycle.rs:497-522`, `:826-827` |
| B3-11 | `SpentUnattributed` (observé par l'expéditeur) | L'expéditeur constate le jeton dépensé sans reçu | — | `payment_spent_without_receipt` | — | Oui (projection) | `payment_lifecycle.rs:810-815`; `CashuBubble.swift:1001-1003`, `:1018-1019` |

### [B4 Affranchissement de relais et récupération](payments/B4-relay-postage-reclaim.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| B4-01 | `NotRequired` | Le relais ne facture pas d'affranchissement | — | — | — | Oui (pour cette livraison) | `PC/transport.rs:79-84` |
| B4-02 | `NotStarted` / `Unavailable` | Pas encore payé, ou paiement impossible | Non payé | — | — | Non | same as above |
| B4-03 | `Created` | Jeton d'affranchissement créé et envoyé au relais avec l'événement | Jeton remis au relais ; le registre du portefeuille enregistre un envoi `relay_stamp` | — | — | Oui (pour cette livraison) | same as above; ledger marker `W/cashu/stamp_reclaim.rs:132-142` |
| B4-04 | Non récupérable (aucun lien) | L'état de la ligne de livraison n'est pas failed / timeout / no_ack / terminal_failed / retry_exhausted, ou un OK a été reçu | — | Aucune entrée affichée | — | — | `A/messaging/stamp_reclaim.rs:5-23` |
| B4-05 | `Available` | Le lien correspond exactement à une saga d'envoi `TokenCreated` | Le jeton n'a peut-être pas encore été encaissé par le relais | « Récupérer » affiché | Récupérer | Non | `W/cashu/stamp_reclaim.rs:128-186` |
| B4-06 | `Processing` | Intention de récupération acceptée (persistée avant de toucher au mint), ou état du mint incertain / injoignable | Intention + preuves d'origine | Traitement | Récupérer à nouveau | Non | `W/cashu/stamp_reclaim.rs:187-199`, `:209-252` |
| B4-07 | `Reclaimed` | Échange terminé | De retour dans le portefeuille | Récupéré | — | Oui | `W/cashu/stamp_reclaim.rs:203-208`, `:415` |
| B4-08 | `Unavailable` | Toutes les preuves dépensées (encaissées par le relais), ou aucune entrée unique correspondante dans le registre | Chez le relais | Non récupérable | — | Oui | `W/cashu/stamp_reclaim.rs:149-158`, `:220-224` |

### [B5 Tickets d'appel payants](payments/B5-paid-call-tickets.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| B5-01 | `preparing` → `Preparing` | L'utilisateur confirme l'achat ; ligne de commande écrite | Jeton pas encore émis ; ou émis mais pas encore écrit | Paiement en cours | Continuer / annuler | Non | `R/app_storage/rtc_tickets.rs:110`; `R/rtc/payments.rs:132` |
| B5-02 | `pending` / `reconciling` → `Confirming` | Jeton écrit dans la commande et remis à notify | Jeton dans la commande | Confirmation en cours ; la page de confirmation n'offre que Continue | Continuer à interroger | Non | `rtc_tickets.rs:128-141`; `R/rtc/payments.rs:58-59`, `:177-199`, `:329` |
| B5-03 | `succeeded` → `Available` / `UsePending` / `Used` | notify a confirmé ; ensuite projeté selon l'usage | Pass | Disponible / en cours d'utilisation / utilisé | Appeler | Oui (côté paiement) | `rtc_tickets.rs:143-153` |
| B5-04 | `failed` → `Failed` | notify a renvoyé un échec, ou le portefeuille n'a pas pu émettre le jeton | Jeton effacé | Échec | — | Oui | `rtc_tickets.rs:118-125`, `:152` |

## C · Messagerie et relations

### [C1 Envoi de messages](messaging/C1-message-send.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C1-01 | `Fetching` | Intention d'aperçu de lien inachevée | Tâche d'envoi | Envoi en cours | Texte / fichier annulables | Non | `A/messaging/delivery_queries.rs:420-424` |
| C1-02 | `Uploading` | Intention de pièce jointe inachevée | Tâche d'envoi + pièces jointes locales | Téléversement en cours | Idem | Non | `delivery_queries.rs:425-429` |
| C1-03 | `Preparing` | La tâche existe, pas encore remise au transport | Tâche d'envoi | Envoi en cours | Idem | Non | `delivery_queries.rs:430-432` |
| C1-04 | `Failed` (dans la tâche) | Téléversement en pause | Tâche d'envoi | Échec | Réessayer | Non | `delivery_queries.rs:421-422` |
| C1-05 | `Sending` | Événements de transport créés ; ni succès métier ni état terminal | `app_message_transport_events` + lignes de livraison par relais | Envoi en cours | Attendre | Non | `delivery_queries.rs:443-445` |
| C1-06 | `Succeeded` | L'unité principale a atteint le succès métier | Lignes de livraison | Envoyé | — | Oui | `delivery_queries.rs:433-437` |
| C1-07 | `Failed` (livraison) | La politique de transport le juge terminal | Lignes de livraison | Échec | Nouvel essai manuel (selon `manual_retryable`) | Oui | `delivery_queries.rs:438-442` |

### [C2 Envoi et téléchargement de pièces jointes](messaging/C2-attachments.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C2-01 | `Remote` | Non téléchargé | URL distante + clé | Téléchargeable | Téléchargement | Non | `A/data_store/message_snapshot.rs:536-540` |
| C2-02 | `Downloading` | Téléchargement lancé (`transfer_state=1`) | Fichier temporaire | Téléchargement en cours | Attendre | Non | `A/app_storage/attachments.rs:356` |
| C2-03 | `Downloaded` | `transfer_state=2` et le fichier local existe | Fichier local | Ouvrable | — | Oui (revient à la projection Remote si le fichier disparaît) | `message_snapshot.rs:527-535` |
| C2-04 | `Failed` | Téléchargement échoué, ou interruption détectée au démarrage / au redémarrage du runtime | `last_error_code` | Échec | Réessayer | Non | `attachments.rs:462-469`, `:472-492` |
| C2-05 | Téléversement en cours | Intention de pièce jointe inachevée | — | `Uploading` | Les messages texte / fichier sont annulables | Non | `A/messaging/delivery_queries.rs:425-429` |
| C2-06 | En pause (échec dans la tâche) | L'échec du téléversement écrit `upload_state='paused'` | — | `Failed` | Nouvel essai permis pour les erreurs réessayables ; pas pour une source manquante, une intention invalide ou une cible non sûre | Non | `A/app_storage/message_sender_attachments.rs:349`; `A/message_sender/projection.rs:73-94` |

### [C3 Demandes d'ami et mise à niveau PQXDH](messaging/C3-friend-requests.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending` (côté expéditeur) | Demande soumise au planificateur de nouvelles tentatives de livraison | Demande en attente + session en attente au niveau du protocole | En attente du correspondant | Retirer | Non | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | Au moins un relais a accepté | Idem | En attente du correspondant | Retirer | Non | `A/types.rs:3512` |
| C3-03 | Acceptée par le correspondant | Réponse du correspondant reçue | Session établie | Contact | — | Oui | — |
| C3-04 | Retirée | L'utilisateur retire ; ou une nouvelle demande au même correspondant la remplace | Session en attente du protocole et transport effacés | — | — | Oui | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | Nouvelle tentative de mise à niveau PQXDH | L'utilisateur déclenche manuellement | — | Messages distincts pour `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` | Réessayer plus tard | Renvoie à chaque appel | `A/types.rs:533-540` |

### [C4 Invitations de groupe et activation du Welcome MLS](messaging/C4-group-invites-mls.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | Invitation envoyée ou reçue ; une nouvelle tentative de livraison revient à pending | — | — | Accepter / refuser / retirer | Non | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | Échec de livraison, ou échec du traitement de la décision | — | — | L'expéditeur MLS peut retenter la livraison (`group_invite_lifecycle_allows_delivery_retry`) | Non | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | Acceptée ; en attente de l'arrivée du message de contrôle d'activation du membre | — | `PendingAcknowledgement` | Attendre | Non | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | Preuve d'activation authentifiée ou OK du relais reçu | — | `Active` | — | Oui | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | Refusée, retirée, expirée | — | — | — | Oui | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | En attente de confirmation pendant la suppression | — | — | — | Non | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

## D · Identité, appareil et récupération

### [D1 Suppression d'identité](identity-device-recovery/D1-identity-deletion.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | L'utilisateur confirme la suppression ; Core écrit le plan de suppression ; les appels actifs de l'identité sont terminés | Identité toujours présente ; tout nouveau travail est bloqué | — | Annuler (uniquement dans cette phase) | Non | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | La plateforme confirme la suppression des identifiants du trousseau | Identifiants supprimés ; données Core en attente de nettoyage | — | Continuer (répétable) | Non | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Points intermédiaires de l'ancien flux de suppression | Idem | — | Continuer ; confirmer la suppression de l'ancien secret | Non | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Suppression terminée | Nettoyage Core terminé ; ligne de suppression retirée | Identité disparue | — | — | Oui | — |

### [D2 Activation de l'appareil, démarrage et restauration de sauvegarde](identity-device-recovery/D2-device-activation-restore.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| D2-01 | `Checking` | Lancement | — | — | Attendre | Non | `A/device_activation.rs:61-62` |
| D2-02 | `SetupRequired` | Aucune donnée | — | — | Créer ou restaurer | Non (en attente de l'utilisateur) | `:63` |
| D2-03 | `ActivationRequired{pending}` | Données présentes, activation non vérifiée | — | — | Activer | Non (en attente de l'utilisateur) | `:64-66` |
| D2-04 | `RecoveryRequired{reason}` | Restauration bloquée, clé de base absente, stockage indisponible / incompatible, trousseau indisponible, identifiants en conflit, politique non prise en charge | — | — | Agir selon le motif | Non (en attente de l'utilisateur) | `:49-58`, `:67-69` |
| D2-05 | `Ready{stopped_reason}` | Tout est prêt ; ou en pause manuelle / comme source de migration / comme cible de migration | — | — | — | Oui (au repos) | `:70-80` |
| D2-06 | `RestoreInspecting` / `RestoreValidating` / `RestoreStaging` (iOS: `preparing` / `prepared`) | — | Zone de préparation ; données d'origine intactes | — | Zone de préparation nettoyée au démarrage | Non | `A/backup_restore.rs:606-621`; `iOS/Services/BackupRestoreTransaction.swift:197-199` |
| D2-07 | `RestoreReplacingLocalData` / `RestoreWritingKeychain` (iOS: `replacement_started` / `restored_data_installed`) | — | Instantané des données d'origine + nouvelles données | — | Retour à l'instantané au démarrage ; le journal est ensuite marqué committed pour éviter un second retour | Non | `BackupRestoreTransaction.swift:200-205`, `:209-258` |
| D2-08 | `RestoreReopeningClient` / `RestoreWalletChecking` (iOS: `validated`) | — | Nouvelles données | — | Idem (retour arrière) | Non | same as above |
| D2-09 | `RestoreComplete` (iOS: `committed`) | — | Nouvelles données ; instantané nettoyé | — | — | Oui | `BackupRestoreTransaction.swift:178-181` |
| D2-10 | `Failed` (iOS: —) | — | Selon le point d'échec | — | Ne peut que revenir à Idle | Oui | `A/backup_restore.rs:698-741` |

### [D3 Récupération du portefeuille](identity-device-recovery/D3-wallet-recovery.md)

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| D3-01 | `Restored` | Toutes les époques terminées | VTXO et sorties récupérés | — | — | Oui | `W/ark/types.rs:69` |
| D3-02 | `NotFound` | Le serveur n'a aucun enregistrement du portefeuille | — | — | Essayer une autre source | Oui | `types.rs:70` |
| D3-03 | `Incomplete` | Une époque incomplète ; le compte reste `recovery_pending`, opérations coopératives refusées, réserve None | Partiellement récupéré | — | Récupérer à nouveau | Non | `W/ark/types.rs:71` |
| D3-04 | `Unavailable` / `IdentityChanged` | Serveur indisponible / l'identité du serveur a changé | — | — | Réessayer plus tard / juger manuellement | Oui (cette tentative) | `types.rs:72-73` |
| D3-05 | Analyse des clés `Ready` / `LimitReached` / `CandidatesMatched` | Analysé jusqu'à la limite ou correspondance par index | Progression de l'analyse | — | Étendre l'analyse | Non | `types.rs:89-108` |
| D3-06 | Analyse de l'historique BTC `Ready` / `LimitReached` / `UnsupportedChainSource` | Idem (BTC partagé) | Adresses découvertes | — | Étendre l'analyse | Non | `types.rs:441-457` |
