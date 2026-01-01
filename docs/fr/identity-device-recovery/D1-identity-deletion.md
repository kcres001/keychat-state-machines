# D1 Suppression d'identité

[Retour au sommaire](../README.md) · D · Identité, appareil et récupération

## Diagramme d'états

GitHub affiche directement le bloc Mermaid ci-dessous ; en cas de problème d'affichage, ouvrez le SVG.

```mermaid
flowchart TD
  U(("L'utilisateur confirme la suppression de l'identité")) --> P1["PlatformCredentialPending<br/>plan de suppression écrit, appels actifs terminés"]
  P1 -->|"[Utilisateur] annuler (uniquement dans cette phase)"| RB["Suppression annulée"]
  P1 -->|"[Plateforme] identifiants du trousseau supprimés"| P2["CoreFinalizePending"]
  P2 -->|"[Local] nettoyage Core terminé"| DONE["Suppression terminée"]
  P2 -.- P2L1(["⟲ [Local] reprendre après interruption"])
  class P2L1 loop
  L(("Ancien flux de suppression")) --> L1["LegacyCorePending"]
  L1 --> L2["LegacyPlatformSecretPending"]
  L2 -->|"[Utilisateur] confirmer la suppression de l'ancien secret"| DONE
  class DONE,RB term
  classDef term fill:#d8f0d8,stroke:#2e7d32,color:#1b3d1b
  classDef user fill:#fff1c4,stroke:#b8860b,color:#4a3800
  classDef gate fill:#e3e8f8,stroke:#3949ab,color:#1a237e
  classDef loop fill:#fafafa,stroke:#777,stroke-dasharray:3 3,color:#333,font-size:12px
```

[SVG](diagrams/D1.svg) · [Source Mermaid](diagrams/D1.mmd)

## Tableau d'états

| ID | État (nom dans le code) | Condition d'entrée | Où sont les fonds / données | Affichage | Ce que l'utilisateur peut faire | Terminal ? | Emplacement dans le code |
|---|---|---|---|---|---|---|---|
| D1-01 | `PlatformCredentialPending` | L'utilisateur confirme la suppression ; Core écrit le plan de suppression ; les appels actifs de l'identité sont terminés | Identité toujours présente ; tout nouveau travail est bloqué | — | Annuler (uniquement dans cette phase) | Non | `A/data_store/identity_deletion.rs:6-50`; rollback `A/app_storage/identities.rs:1386-1409` |
| D1-02 | `CoreFinalizePending` | La plateforme confirme la suppression des identifiants du trousseau | Identifiants supprimés ; données Core en attente de nettoyage | — | Continuer (répétable) | Non | `identity_deletion.rs:54-82`, `:101-138` |
| D1-03 | `LegacyCorePending` / `LegacyPlatformSecretPending` | Points intermédiaires de l'ancien flux de suppression | Idem | — | Continuer ; confirmer la suppression de l'ancien secret | Non | `identity_deletion.rs:121-126`, `:237`, `:271` |
| D1-04 | Suppression terminée | Nettoyage Core terminé ; ligne de suppression retirée | Identité disparue | — | — | Oui | — |

« Idem » renvoie à la ligne précédente du tableau. Dans la colonne d'affichage, les valeurs `xxx.yyy` sont des clés de localisation.
