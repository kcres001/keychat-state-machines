# C3 Demandes d'ami et mise à niveau PQXDH

[Retour au sommaire](../README.md) · C · Messagerie et relations

## Diagramme d'états

GitHub affiche directement le bloc Mermaid ci-dessous ; en cas de problème d'affichage, ouvrez le SVG.

```mermaid
flowchart TD
  U(("L'utilisateur envoie une demande d'ami")) --> P["Pending<br/>dans le planificateur de nouvelles tentatives"]
  P -->|"[Relais] acceptée"| PB["Published"]
  P -->|"[Utilisateur] retirer"| C["Retirée"]
  PB -->|"[Utilisateur] retirer"| C
  P -->|"[Utilisateur] nouvelle demande au même correspondant"| SP["Remplacée"]
  PB -->|"[Correspondant] accepte"| OK["Contact"]
  PB -.- PBL1(["⟲ [Correspondant] pas de réponse → attendre"])
  class PBL1 loop
  class OK,C,SP term
  class PB user
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/C3.svg) · [Source Mermaid](diagrams/C3.mmd)

## Tableau d'états

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| C3-01 | `Pending` (côté expéditeur) | Demande soumise au planificateur de nouvelles tentatives de livraison | Demande en attente + session en attente au niveau du protocole | En attente du correspondant | Retirer | Non | `A/friend_request.rs:1260-1263`, `:1611-1625` |
| C3-02 | `Published` | Au moins un relais a accepté | Idem | En attente du correspondant | Retirer | Non | `A/types.rs:3512` |
| C3-03 | Acceptée par le correspondant | Réponse du correspondant reçue | Session établie | Contact | — | Oui | — |
| C3-04 | Retirée | L'utilisateur retire ; ou une nouvelle demande au même correspondant la remplace | Session en attente du protocole et transport effacés | — | — | Oui | `A/friend_request.rs:1265-1369`, `:1220-1258` |
| C3-05 | Nouvelle tentative de mise à niveau PQXDH | L'utilisateur déclenche manuellement | — | Messages distincts pour `Sent` / `AlreadyUpgraded` / `PeerNotReady` / `IneligibleRoom` / `InFlight` | Réessayer plus tard | Renvoie à chaque appel | `A/types.rs:533-540` |

« Idem » renvoie à la ligne précédente du tableau. Dans la colonne d'affichage, les valeurs `xxx.yyy` sont des clés de localisation.
