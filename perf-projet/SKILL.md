---
name: perf-projet
description: >
  Optimise les performances d'un projet web de bout en bout : instrumente,
  établit le verdict rapide/lent, puis prend en charge les 30 optimisations
  classiques (squelettes, webp/avif, dédoublonnage d'appels, debounce,
  pagination, CDN, file de jobs, N+1, compression, Redis, dépendances inutiles,
  timeouts, streaming IA, UI optimiste, lazyload, requêtes parallèles, index,
  colocalisation DB, SSR/SSG, scripts tiers différés, découpage JS, virtualisation
  de listes, colonnes utiles, pool de connexions, re-renders, cache navigateur,
  load balancer, polices, RUM, devices lents) — il les VÉRIFIE une par une et
  applique le CORRECTIF quand une mesure le justifie. À déclencher quand
  l'utilisateur veut « optimiser mon site », « mon site est lent », « auditer la
  perf », « appliquer les optimisations », « vérifier/patcher la perf », « mettre
  en place le budget perf », ou fournit un projet à accélérer. Ne patche que ce
  qu'une mesure désigne.
---

# perf-projet — vérifier et patcher les performances

Pendant exécutable du référentiel [../regles-perf.md](../regles-perf.md), et
jumeau côté perf de `secu-projet`. Il prend en charge **les 30 directives**
listées par l'utilisateur : pour chacune il établit l'**état** (déjà faite / à
faire / inutile ici), et **applique le correctif quand — et seulement quand — une
mesure le désigne**.

## La règle qui commande tout : pas de patch sans mesure

C'est la règle de l'utilisateur, non négociable. Un correctif qu'aucune mesure
ne justifie n'est pas une amélioration, c'est de la dette : Redis sans hit ratio
connu, un index sans `EXPLAIN`, une file de jobs sans profil. Donc l'ordre est
toujours :

```
N0 instrumenter → N1 verdict → profiler le pire parcours → patcher ce que le
profil désigne, un correctif à la fois, gain mesuré → N5 verrouiller en CI
```

Démarrer au patch sans mesure est le mode d'échec que ce skill refuse. Si
l'instrumentation manque (étape 1), il la pose d'abord ou s'arrête là.

## Les trois états d'une directive

Pour chacune des 30, le skill tranche :

- **VÉRIFIÉE OK** — déjà en place, preuve à l'appui. On ne touche pas.
- **À PATCHER** — manquante *et* une mesure la désigne sur un parcours réel. On
  applique le correctif, puis on remesure.
- **NON JUSTIFIÉE ICI** — manquante mais aucune mesure ne la réclame, ou
  contexte inadapté. On ne l'applique pas, et on écrit pourquoi (c'est une
  décision, pas un oubli).

Une directive n'est jamais « patchée par principe ». Voir les anti-règles.

## Marche à suivre

### Étape 0 — Cartographier le projet
Pile technique, framework, base, hébergeur, CI, CDN/WAF existants.
`grep`/`glob` plutôt que tout lire. Identifier le **parcours le plus fréquenté**
et le **plus lent ressenti** : c'est sur lui que tout se joue.

### Étape 1 — Instrumenter (bloquant)
RUM (LCP/INP/CLS/TTFB), APM/tracing (p50/p95/p99 par endpoint), slow query log,
hit ratio des caches. Détail : N0 de [../regles-perf.md](../regles-perf.md). Sans
ça, aucun patch n'est justifiable — le skill s'arrête ici et le dit.

### Étape 2 — Verdict
Remplir N1 en **p75**, segmenté (page, pays, device, auth). Nommer le pire
parcours. Le labo diagnostique (Lighthouse), le terrain arbitre (RUM).

### Étape 3 — Profiler ce parcours
Waterfall réseau + chaîne critique, trace APM span par span, `EXPLAIN` des
requêtes impliquées, profil du fil principal. Sortie : une **liste ordonnée des
coûts en ms**, pas une liste d'optimisations possibles. C'est cette liste qui
décide quelles directives passent en « À PATCHER ».

### Étape 4 — Prendre en charge les 30 directives
Pour chaque directive que le profil désigne : vérifier l'état, appliquer le
correctif, **remesurer** et consigner le gain. Une à la fois. La table complète
— vérification, correctif, et la mesure qui l'autorise — est dans
[references/directives.md](references/directives.md).

### Étape 5 — Verrouiller (N5)
Budget perf en CI qui fait échouer la PR (poids de bundle, score de lab),
alerte sur dégradation du p75, et le journal des correctifs. Sans ça, les 30
points sont refaits dans six mois.

## Livrables
- Fiche projet + verdict N1 segmenté.
- La **liste ordonnée des coûts en ms** du parcours critique.
- Le **tableau des 30** avec l'état de chacune (OK / patchée / non justifiée +
  raison).
- Les **correctifs appliqués**, chacun avec gain avant/après chiffré.
- Le **budget perf en CI**.

## Anti-règles — ce qu'on ne patche jamais à l'aveugle
- **Redis** avant de connaître le hit ratio attendu → une base de plus à
  maintenir et une classe de bugs d'invalidation.
- **File de jobs** avant d'avoir profilé → de l'asynchrone pour masquer une
  requête mal écrite.
- **Index** avant `EXPLAIN` → écritures ralenties, lectures inchangées.
- **CDN** avant d'avoir mesuré le TTFB → un cache devant un serveur lent.
- **Micro-optimisations de rendu** avant de regarder le réseau → 5 ms gagnés sur
  2 s d'attente.
- **Squelettes** présentés comme un gain de vitesse → c'est du perçu, légitime,
  mais ça ne déplace aucune métrique de N1.
