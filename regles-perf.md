# Référentiel perf — règles de contrôle d'un projet

Version 1 — 2026-11-04

Usage : grille d'audit d'un projet web existant, et contrat de non-régression
pour la suite. Chaque règle a un **identifiant**, un **seuil chiffré**, et une
**preuve** attendue. Sans preuve, la règle est NON VÉRIFIÉE — jamais « OK ».

---

## Les 3 lois

1. **Pas de remède sans mesure.** Une optimisation qu'aucune mesure ne désigne
   est de la dette déguisée en travail.
2. **Le p75 décide, pas la moyenne.** Une moyenne saine cache systématiquement
   le quart d'utilisateurs qui part.
3. **Ce qui n'est pas gardé en CI revient.** Toute règle sans porte automatique
   sera reperdue au prochain trimestre.

**Ordre d'application, non négociable :** N0 (instrumenter) → N1 (verdict) →
N2/N3/N4 (corriger ce que la mesure désigne) → N5 (verrouiller).
Démarrer en N2 sans N0 est le mode d'échec le plus courant.

---

## N0 — Instrumentation (bloquant : rien ne se mesure sans ça)

| ID | Règle | Preuve attendue |
|----|-------|-----------------|
| N0.1 | Un **RUM** collecte LCP / INP / CLS / TTFB sur le trafic réel | tableau de bord avec données < 7 jours |
| N0.2 | Les métriques sont **segmentables** par type de page, pays, device, navigateur, authentifié ou non | capture d'un filtre appliqué |
| N0.3 | Un **APM / tracing** donne la latence par endpoint en p50/p95/p99 | trace d'une requête lente, span par span |
| N0.4 | Le **slow query log** de la base est activé, seuil ≤ 100 ms | 10 requêtes les plus lentes, avec leur `EXPLAIN` |
| N0.5 | Les **erreurs et timeouts** sont comptés, pas seulement la latence | taux d'erreur par endpoint sur 7 jours |
| N0.6 | Les **hit ratio** de chaque cache (CDN, Redis, navigateur) sont exposés | 3 chiffres datés |

> Un cache dont on ne connaît pas le hit ratio n'est pas un cache, c'est une
> croyance.

---

## N1 — Verdict : le site est-il lent ?

Mesuré en **p75 sur 28 jours**, trafic réel, pas en laboratoire.

| ID | Métrique | Rapide | À corriger | Lent |
|----|----------|--------|-----------|------|
| N1.1 | LCP | < 2,5 s | 2,5–4 s | > 4 s |
| N1.2 | INP | < 200 ms | 200–500 ms | > 500 ms |
| N1.3 | CLS | < 0,1 | 0,1–0,25 | > 0,25 |
| N1.4 | TTFB | < 200 ms | 200–600 ms | > 600 ms |
| N1.5 | API p95 (endpoints de rendu) | < 300 ms | 300–1000 ms | > 1 s |
| N1.6 | API p99 | < 1 s | 1–3 s | > 3 s |
| N1.7 | Taux d'erreur 5xx | < 0,1 % | 0,1–1 % | > 1 % |
| N1.8 | Profondeur de la chaîne critique | ≤ 3 | 4–5 | > 5 |
| N1.9 | JS exécuté au chargement | < 150 ko gz | 150–350 ko | > 350 ko |

**Conditions de mesure à documenter avec tout chiffre** (sinon le chiffre ne
vaut rien) : cache froid ou chaud, bridage réseau (référence : 4G lente,
~1,6 Mb/s, 150 ms RTT), device (référence : milieu de gamme Android
~4× plus lent qu'un laptop), position géographique du test, authentifié ou non.

| ID | Règle |
|----|-------|
| N1.10 | Tout chiffre perf communiqué cite ses conditions de mesure |
| N1.11 | Le **laboratoire diagnostique, le terrain arbitre** : Lighthouse ne sert jamais de verdict, seulement à trouver la cause |
| N1.12 | Le parcours le plus lent et le plus fréquenté est identifié nommément, avant toute correction |

---

## N2 — Front

| ID | Règle | Seuil / preuve |
|----|-------|----------------|
| N2.1 | Images servies en **AVIF ou WebP**, dimensionnées à l'affichage réel | aucune image > 2× sa taille affichée |
| N2.2 | Images sous la ligne de flottaison en **lazy load** ; celle du LCP jamais en lazy | audit DOM |
| N2.3 | **Dimensions réservées** pour tout média et encart (pub, bannière) | CLS < 0,1 |
| N2.4 | JS **découpé par route**, pas de bundle unique | N1.9 respecté |
| N2.5 | **Scripts tiers** chargés après la page, et chacun justifié par un propriétaire nommé | liste des tiers, avec leur coût mesuré en ms |
| N2.6 | Pages rendues **au build ou côté serveur** quand le contenu le permet | liste des routes et de leur mode de rendu |
| N2.7 | **Polices auto-hébergées**, 2 graisses maximum, `font-display: swap` | poids total des polices < 100 ko |
| N2.8 | Listes > 100 éléments **virtualisées** ou paginées | aucune liste à rendu complet au-delà du seuil |
| N2.9 | Aucun **re-render** non justifié sur interaction courante | profil React/Vue avant-après |
| N2.10 | **Recherche débattue** (300 ms) et requête annulable | une frappe de 10 caractères = 1 requête |
| N2.11 | Appels API **dédoublonnés** : une même clé en vol n'est pas relancée | onglet réseau sur un parcours type |
| N2.12 | Requêtes indépendantes **en parallèle**, jamais en cascade par confort d'écriture | waterfall sans escalier évitable |
| N2.13 | **Squelettes de chargement** sur tout écran > 300 ms — perçu, pas mesuré : ne compte jamais comme une optimisation |
| N2.14 | **Mise à jour optimiste** de l'interface sur les actions sûres et réversibles |
| N2.15 | Réponses d'IA ou de flux **streamées token par token** | premier token < 1 s |
| N2.16 | Cache navigateur : statiques **immuables** et versionnés, HTML non cachable | en-têtes `Cache-Control` relevés |

---

## N3 — Backend et base de données

| ID | Règle | Seuil / preuve |
|----|-------|----------------|
| N3.1 | Aucune requête **N+1** sur un parcours mesuré | compte de requêtes par endpoint, avant/après |
| N3.2 | **Index** sur les colonnes filtrées et triées fréquemment — posé après `EXPLAIN`, jamais à l'intuition | plan d'exécution sans scan complet sur table > 10 k lignes |
| N3.3 | **Colonnes sélectionnées explicitement**, pas de `SELECT *` sur les chemins chauds |
| N3.4 | **Pool de connexions** dimensionné et surveillé | taux de saturation du pool < 70 % en pointe |
| N3.5 | Résultats d'API **paginés**, avec plafond serveur | aucune réponse non bornée |
| N3.6 | Taille de réponse d'API | p95 < 100 ko |
| N3.7 | **Timeout explicite** sur tout appel sortant, plus un repli | aucun appel sans timeout dans le code |
| N3.8 | **Disjoncteur** (circuit breaker) sur les dépendances externes critiques |
| N3.9 | Tâche > 1 s déportée en **file de jobs**, avec reprise et file d'échecs |
| N3.10 | **Données chaudes en cache** (Redis) — uniquement si hit ratio mesuré > 80 %, et avec invalidation écrite |
| N3.11 | **Compression** gzip ou brotli sur tout texte > 1 ko |
| N3.12 | **Dépendances inutiles supprimées** | inventaire daté, poids avant/après |

---

## N4 — Infrastructure et réseau

| ID | Règle | Seuil / preuve |
|----|-------|----------------|
| N4.1 | Statiques servis par un **CDN**, hit ratio > 90 % |
| N4.2 | **Serveurs applicatifs proches de la base** | latence app↔DB < 5 ms |
| N4.3 | **Répartition de charge** sur ≥ 2 instances, et bascule testée |
| N4.4 | **Démarrages à froid** mesurés si serverless | p95 < 500 ms |
| N4.5 | **Test de charge** passé : le site tient la pointe attendue × 2 | rapport k6/Locust avec latence et erreurs sous charge |
| N4.6 | **Saturation** (CPU, mémoire, connexions) sous les 70 % en pointe |
| N4.7 | Le comportement **en dégradation** est défini : qu'est-ce qui tombe en premier, et comment |

> N4.5 est la règle la plus souvent absente. Tout le reste du référentiel
> décrit un site à un utilisateur ; celle-ci est la seule qui parle du jour où
> il y en a deux cents.

---

## N5 — Non-régression (sans ça, l'audit a une durée de vie de 6 mois)

| ID | Règle | Preuve |
|----|-------|--------|
| N5.1 | **Budget perf en CI** : poids de bundle et métriques de lab, qui **fait échouer** la PR | une PR rouge pour dépassement |
| N5.2 | **Alerte** sur dégradation du p75 au-delà de 10 %, vers un destinataire nommé |
| N5.3 | **Un changement perf à la fois**, mesuré avant/après | journal des optimisations, gain chiffré par entrée |
| N5.4 | Toute optimisation sans gain mesuré est **retirée**, pas conservée par principe |
| N5.5 | Les seuils de N1 sont **écrits dans le dépôt** et revus à chaque trimestre |
| N5.6 | Un parcours critique est **testé sur device lent** à chaque version majeure |

---

## Anti-règles — ce qu'on n'applique pas sans mesure préalable

Chacune est utile au bon moment, et du coût pur au mauvais :

- **Redis** avant de connaître le hit ratio attendu → une base de données de
  plus à maintenir, et une classe entière de bugs d'invalidation.
- **File de jobs** avant d'avoir profilé → asynchrone pour masquer une requête
  mal écrite.
- **Index** avant `EXPLAIN` → écritures ralenties, lectures inchangées.
- **CDN** avant d'avoir mesuré le TTFB → cache devant un serveur lent.
- **Micro-optimisations de rendu** avant de regarder le réseau → 5 ms gagnés
  sur 2 s d'attente.
- **Squelettes** présentés comme un gain de vitesse → c'est du perçu, et c'est
  légitime, mais ça ne déplace aucune des métriques de N1.

---

## Procédure d'audit (une demi-journée)

1. **Instrumenter** — N0 complet. Si N0 manque, l'audit s'arrête ici : rien
   d'autre n'est conclusif.
2. **Verdict** — remplir N1 en p75, segmenté. Nommer le pire parcours fréquenté.
3. **Diagnostiquer** — sur ce parcours : waterfall réseau, trace APM,
   `EXPLAIN` des requêtes impliquées, profil du fil principal. Produire une
   **liste ordonnée des coûts en ms**, pas une liste d'améliorations possibles.
4. **Corriger** — uniquement les règles N2/N3/N4 que la liste du point 3
   désigne. Une à la fois, gain mesuré à chaque fois.
5. **Charger** — rejouer N4.5 après correction.
6. **Verrouiller** — N5, puis réaudit à 3 mois.

---

## Fiche de restitution

```
Projet :                          Date :              Auditeur :
Conditions de mesure :            (cache, réseau, device, géo, auth)

VERDICT        LCP ___   INP ___   CLS ___   TTFB ___   API p95 ___
               → rapide / à corriger / lent

N0 instrumentation    ___ / 6 vérifiées
N2 front              ___ / 16
N3 backend            ___ / 12
N4 infra              ___ / 7
N5 non-régression     ___ / 6

Les 3 coûts les plus lourds, en ms, sur le parcours ______ :
  1.  ___ ms  →  règle ____
  2.  ___ ms  →  règle ____
  3.  ___ ms  →  règle ____

Règles NON VÉRIFIABLES faute d'instrumentation :
Prochaine mesure prévue le :
```

---

## Correspondance avec la liste d'origine

Les 28 points de départ se retrouvent en N2/N3/N4. Ce que le référentiel
ajoute : N0 en entier, N1 en entier, N4.4 à N4.7, N5 en entier, et les
anti-règles. C'est-à-dire tout ce qui permet de répondre à « est-ce lent ? »
plutôt qu'à « comment accélérer ? ».
