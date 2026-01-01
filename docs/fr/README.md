# Machines à états de Keychat : diagrammes et tableaux d'états

[English](../en/README.md) · [Español](../es/README.md) · **Français** · [日本語](../ja/README.md) · [简体中文](../zh-Hans/README.md) · [繁體中文](../zh-Hant/README.md)

Ce document décrit les flux en plusieurs étapes de Keychat qui peuvent rester en attente ou impliquent des fonds : les états de chaque flux, comment on y entre, où se trouvent les fonds ou les données, ce qu'affiche l'interface, ce que l'utilisateur peut faire, quels états sont terminaux et qui déclenche chaque transition.

Base de code : Keychat iOS 1.5, commit `97c69dee1`. Tous les numéros de ligne renvoient à ce commit.

## Comment lire

### Légende

- **Vert**: État terminal
- **Jaune**: En attente de l'utilisateur, ou récupérable
- **Losange bleu-violet**: Pré-vérification / décision
- **Cadre arrondi en pointillés ⟲**: Boucle qui reste dans le même état (nouvel essai, attente, mise en attente) ; l'étiquette indique le déclencheur
- **Flèche pleine**: Transition ; le préfixe entre crochets indique le déclencheur (utilisateur, serveur, chaîne / source de chaîne, correspondant, mint, relais, notify, maintenance, reprise au démarrage, local)
- **Flèche en pointillés**: Transition qui ne se produit que sous une condition précise

« Idem » renvoie à la ligne précédente du tableau. Dans la colonne d'affichage, les valeurs `xxx.yyy` sont des clés de localisation.

### Abréviations de chemins

- `W/` = `rust/keychat-wallet/src/`
- `BK/` = `rust/patches/bark/bark/src/`
- `A/` = `rust/keychat-app-core/src/`
- `R/` = `rust/keychat-app-core/src/`
- `PC/` = `rust/keychat-protocol-core/src/`
- `WU/` = `rust/keychat-wallet-uniffi/src/`
- `iOS/` = `apps/ios/Keychat/`

Un `:ligne` isolé dans une cellule renvoie au fichier précédent de la même cellule.

## Sommaire

### A · Fonds on-chain et Ark

- [A1 Dépôt vers Ark, envoi direct de BTC, accélération RBF](ark-onchain/A1-onboard-direct-send-rbf.md)
- [A2 Envoi de BTC depuis Ark vers une adresse on-chain](ark-onchain/A2-ark-to-onchain-send.md)
- [A3 Réserve de frais de sortie](ark-onchain/A3-exit-fee-reservation.md)
- [A4 Sortie forcée et réclamation](ark-onchain/A4-force-exit-and-claim.md)
- [A5 Rafraîchissement et maintenance automatique](ark-onchain/A5-refresh-and-maintenance.md)

### B · Paiements et encaissements

- [B1 Envoi et réception Lightning (Ark)](payments/B1-lightning-ark.md)
- [B2 Réception Cashu](payments/B2-cashu-receive.md)
- [B3 Virements dans le chat, enveloppes rouges et reçus](payments/B3-chat-transfer-red-packet.md)
- [B4 Affranchissement de relais et récupération](payments/B4-relay-postage-reclaim.md)
- [B5 Tickets d'appel payants](payments/B5-paid-call-tickets.md)

### C · Messagerie et relations

- [C1 Envoi de messages](messaging/C1-message-send.md)
- [C2 Envoi et téléchargement de pièces jointes](messaging/C2-attachments.md)
- [C3 Demandes d'ami et mise à niveau PQXDH](messaging/C3-friend-requests.md)
- [C4 Invitations de groupe et activation du Welcome MLS](messaging/C4-group-invites-mls.md)

### D · Identité, appareil et récupération

- [D1 Suppression d'identité](identity-device-recovery/D1-identity-deletion.md)
- [D2 Activation de l'appareil, démarrage et restauration de sauvegarde](identity-device-recovery/D2-device-activation-restore.md)
- [D3 Récupération du portefeuille](identity-device-recovery/D3-wallet-recovery.md)

- [Tous les états (tableau unique)](all-states.md)
