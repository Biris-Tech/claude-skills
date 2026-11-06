# Pipeline CI de sécurité

Les portes qui font **échouer une PR**. Une porte sans seuil bloquant est
décorative. Toutes ces étapes sont passives (lecture de code et d'artefacts) —
elles ne touchent aucun serveur — sauf le DAST léger, qui vise la préprod
autorisée.

## Les cinq portes + la sixième qui compte le plus

| Porte | Commande | Seuil bloquant |
|-------|----------|----------------|
| Secrets | `gitleaks detect --source . --log-opts="--all" --redact` | tout résultat = échec |
| Code (SAST) | `semgrep ci --config p/security-audit --config p/secrets` | sévérité ≥ ERROR = échec |
| Dépendances | `osv-scanner -r .` (ou `npm audit --audit-level=high`) | CVE ≥ HIGH = échec |
| Conteneur | `trivy image $IMAGE` puis `trivy config .` | CVE ≥ HIGH corrigeable = échec |
| DAST léger | `zap-baseline.py -t https://staging -c rules.conf` | alerte confirmée = échec |
| **Autorisations** | la suite de tests du projet | un test rouge = échec |

Le point sur `gitleaks` : `--log-opts="--all"` couvre **tout l'historique et
toutes les branches**. Un secret commité puis retiré reste publié — il faut le
**tourner**, pas se contenter de le supprimer du dernier commit.

## Exemple GitHub Actions

À adapter à l'écosystème réel (langage, gestionnaire de paquets, registre).

```yaml
name: securite
on: [pull_request]

permissions:
  contents: read

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # historique complet, indispensable
      - uses: gitleaks/gitleaks-action@v2

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with: { config: "p/security-audit p/secrets" }

  deps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google/osv-scanner-action@v1
        with: { scan-args: "-r ." }

  conteneur:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:ci .
      - uses: aquasecurity/trivy-action@master
        with: { image-ref: "app:ci", severity: "HIGH,CRITICAL", exit-code: "1" }
      - uses: aquasecurity/trivy-action@master
        with: { scan-type: "config", severity: "HIGH,CRITICAL", exit-code: "1" }

  tests-autorisation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test-secu    # cf. batterie-tests.md, section 1
```

Le DAST (`zap-baseline`) tourne séparément contre la préprod, pas sur chaque PR
(trop lent) — plutôt en nocturne ou avant un déploiement.

## Durcir le pipeline lui-même

La chaîne de construction est une cible : un secret volé en CI ouvre tout.

- `permissions:` au minimum (lecture par défaut, écriture accordée au cas par cas).
- Actions **épinglées à un SHA**, pas à une étiquette mobile (`@v4` peut être
  reciblé par l'auteur).
- Secrets injectés par le coffre du CI, jamais en clair, jamais lisibles depuis
  une PR d'un fork.
- Protection de branche : revue obligatoire, portes vertes exigées, personne ne
  pousse seul en production.
- Scripts de post-installation des dépendances surveillés (vecteur de
  compromission de chaîne d'approvisionnement).

## Remonter l'alerte, pas seulement bloquer

Agréger les résultats au format SARIF vers l'onglet sécurité du dépôt, pour
suivre l'évolution plutôt que de traiter chaque PR en isolation.
