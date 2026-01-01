# C4 Invitations de groupe et activation du Welcome MLS

[Retour au sommaire](../README.md) · C · Messagerie et relations

## Diagramme d'états

GitHub affiche directement le bloc Mermaid ci-dessous ; en cas de problème d'affichage, ouvrez le SVG.

```mermaid
flowchart TD
  U(("Invitation envoyée / reçue")) --> P["pending"]
  P -->|"[Relais] échec de livraison"| F["failed"]
  F -->|"[Expéditeur MLS] retenter la livraison"| P
  P -->|"[Utilisateur] accepter"| A["accepted<br/>Welcome: PendingAcknowledgement"]
  A -->|"[Local] envoyer le message de contrôle d'activation"| FN["finalizing"]
  FN -->|"[OK du relais / preuve authentifiée du correspondant]"| EN["enabled<br/>Welcome: Active"]
  A -.- AL1(["⟲ [Local] nouvelle tentative d'activation"])
  class AL1 loop
  P -->|"[Utilisateur] refuser"| DC["declined"]
  P -->|"[Utilisateur] retirer"| CN["canceled"]
  P -->|"[Temps] expirée"| EX["expired"]
  P -->|"[Utilisateur] supprimer"| RP["removePending"]
  RP -->|"[Récupération] suppression confirmée"| DC
  class EN,DC,CN,EX term
  class F user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C4.svg) · [Source Mermaid](diagrams/C4.mmd)

## Tableau d'états

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C4-01 | `pending` | Invitation envoyée ou reçue ; une nouvelle tentative de livraison revient à pending | — | — | Accepter / refuser / retirer | Non | `A/group_invite/outgoing.rs:216`, `:395`; `A/group_invite/welcome.rs:212`; `A/group_invite/delivery.rs:413` |
| C4-02 | `failed` | Échec de livraison, ou échec du traitement de la décision | — | — | L'expéditeur MLS peut retenter la livraison (`group_invite_lifecycle_allows_delivery_retry`) | Non | `outgoing.rs:440`, `:482`, `:519`; `A/group_invite/decisions.rs:244`; `A/group_invite.rs:308-318` |
| C4-03 | `accepted` / `finalizing` | Acceptée ; en attente de l'arrivée du message de contrôle d'activation du membre | — | `PendingAcknowledgement` | Attendre | Non | `A/group_invite/projection.rs:14`, `:24` |
| C4-04 | `enabled` | Preuve d'activation authentifiée ou OK du relais reçu | — | `Active` | — | Oui | `A/group_invite/delivery.rs:513`; `projection.rs:25` |
| C4-05 | `declined` / `canceled` / `expired` | Refusée, retirée, expirée | — | — | — | Oui | `A/group_invite/removal_recovery.rs:55-63`, `:93-103` |
| C4-06 | `removePending` | En attente de confirmation pendant la suppression | — | — | — | Non | `A/group_invite/decisions.rs:175`; `removal_recovery.rs:235`, `:345` |

« Idem » renvoie à la ligne précédente du tableau. Dans la colonne d'affichage, les valeurs `xxx.yyy` sont des clés de localisation.
