# Les 30 directives — vérifier, patcher, sous condition

Reprend **une par une** les 30 consignes de départ. Colonnes :
- **Vérif** : comment établir l'état actuel (la preuve).
- **Patch** : le correctif à appliquer.
- **Mesure qui l'autorise** : la condition sans laquelle on ne patche pas — la
  règle « pas de patch sans mesure ». Si la condition n'est pas réunie →
  NON JUSTIFIÉE ICI.

Les correctifs se décrivent indépendamment de la pile ; au moment d'appliquer,
les adapter au framework réel du projet (étape 0 du SKILL.md).

---

## Front — chargement et rendu

**1. Écrans de chargement squelette**
- Vérif : écran blanc > 300 ms sur un parcours ? capture au ralenti réseau.
- Patch : squelette sur les zones qui arrivent après 300 ms.
- Mesure : un écran mesuré > 300 ms. ⚠ perçu seulement — **ne compte pas**
  comme gain sur LCP/INP, à consigner comme tel.

**2. Images en WebP ou AVIF**
- Vérif : `curl -sI image | grep content-type` + poids vs taille affichée.
- Patch : conversion AVIF/WebP, `srcset` dimensionné, repli.
- Mesure : images dans la chaîne critique, ou poids image > 2× la taille
  affichée.

**15. Lazyload des images sous la ligne de flottaison**
- Vérif : audit DOM — `loading="lazy"` présent hors écran, **absent** sur
  l'image du LCP.
- Patch : `loading="lazy"` sous la flottaison ; jamais sur le LCP.
- Mesure : images hors écran chargées au départ dans le waterfall.

**19. Rendu côté serveur ou au build (SSR/SSG)**
- Vérif : liste des routes et de leur mode de rendu ; TTFB utile vs écran blanc.
- Patch : SSG pour le contenu stable, SSR pour le dynamique, surveiller le coût
  d'hydratation.
- Mesure : contenu critique qui attend le JS (LCP tardif en N1).

**20. Scripts tiers après la page**
- Vérif : waterfall — tiers bloquants ? liste des tiers avec coût en ms.
- Patch : `async`/`defer`, chargement différé, chaque tiers justifié par un
  propriétaire.
- Mesure : un tiers mesuré dans le blocage du fil principal.

**21. Découper le JS par page**
- Vérif : taille du bundle d'entrée ; N1.9 (< 150 ko gz).
- Patch : découpage par route, imports dynamiques.
- Mesure : JS d'entrée > seuil, ou code non utilisé sur la route.

**22. Virtualiser les longues listes**
- Vérif : listes > 100 éléments rendues en entier ? compte de nœuds DOM.
- Patch : fenêtrage (virtualisation) ou pagination.
- Mesure : liste > 100 éléments causant jank/INP élevé.

**25. Éviter les re-renders inutiles**
- Vérif : profil React/Vue sur une interaction courante.
- Patch : mémoïsation, dépendances stables, découpage d'état — ciblé.
- Mesure : re-render coûteux **mesuré** au profiler (jamais « au cas où »).

**28. Héberger les polices, limiter les graisses**
- Vérif : polices tierces bloquantes ? nombre de graisses ? poids total.
- Patch : auto-hébergement, 2 graisses max, `font-display: swap`, préchargement.
- Mesure : polices dans la chaîne critique ou > 100 ko.

---

## Front — réseau et interactions

**3. Bloquer les appels API en double**
- Vérif : onglet réseau sur un parcours — même requête plusieurs fois ?
- Patch : déduplication des requêtes en vol (clé = méthode+URL+corps).
- Mesure : doublons observés dans le waterfall.

**4. Debounce sur la recherche**
- Vérif : taper 10 caractères — combien de requêtes partent ?
- Patch : debounce ~300 ms + annulation de la requête précédente.
- Mesure : > 1 requête pour une frappe continue.

**13. Streamer les réponses d'IA par token**
- Vérif : temps jusqu'au premier token sur une génération.
- Patch : streaming token par token (SSE/flux).
- Mesure : réponse IA affichée d'un bloc, premier token > 1 s.

**14. Mettre à jour l'UI avant le serveur (optimiste)**
- Vérif : délai entre clic et retour visuel sur une action sûre.
- Patch : mise à jour optimiste + rollback sur échec — **actions réversibles
  seulement**.
- Mesure : latence perçue sur une action réversible fréquente.

**16. Requêtes indépendantes en parallèle**
- Vérif : waterfall — escalier de requêtes sans dépendance entre elles ?
- Patch : paralléliser (`Promise.all` / équivalent).
- Mesure : cascade évitable visible dans le waterfall.

**26. Cache navigateur**
- Vérif : en-têtes `Cache-Control` des statiques et du HTML.
- Patch : statiques immuables versionnés (long max-age), HTML non cachable.
- Mesure : statiques re-téléchargés à chaque visite (N0.6 / waterfall cache
  chaud).

---

## Backend et base de données

**5. Paginer les résultats de l'API**
- Vérif : une liste renvoie-t-elle tout ? taille de réponse p95.
- Patch : pagination (curseur de préférence) + plafond serveur.
- Mesure : réponse non bornée, ou p95 > 100 ko (N3.6).

**8. Corriger les requêtes N+1**
- Vérif : compte de requêtes SQL par endpoint (log/APM).
- Patch : chargement groupé (eager/`JOIN`/dataloader).
- Mesure : N+1 identifié sur un endpoint du parcours critique.

**17. Index sur les données fréquemment utilisées**
- Vérif : `EXPLAIN` des requêtes lentes — scan complet ?
- Patch : index sur colonnes filtrées/triées, **après** `EXPLAIN`.
- Mesure : scan séquentiel sur table > 10 k lignes dans le plan (jamais à
  l'intuition — anti-règle).

**23. Sélectionner seulement les colonnes utiles**
- Vérif : `SELECT *` sur les chemins chauds ?
- Patch : projection explicite des colonnes.
- Mesure : `SELECT *` sur un endpoint chaud, surtout avec colonnes lourdes.

**24. Pool de connexions**
- Vérif : pool configuré ? taux de saturation en pointe.
- Patch : pool dimensionné et surveillé.
- Mesure : connexions ouvertes/fermées par requête, ou saturation > 70 % (N3.4).

**7. File de jobs pour les tâches lentes**
- Vérif : tâche synchrone > 1 s dans une requête (APM) ?
- Patch : file de jobs avec reprise + file d'échecs.
- Mesure : tâche > 1 s profilée (anti-règle : jamais avant d'avoir profilé).

**9. Compression gzip (ou brotli)**
- Vérif : `curl -H 'Accept-Encoding: br,gzip' -sI` — en-tête de compression ?
- Patch : compression sur tout texte > 1 ko.
- Mesure : réponses texte non compressées dans le waterfall.

**10. Cacher les données chaudes dans Redis**
- Vérif : quelles données relues souvent et coûteuses à calculer ? hit ratio
  attendu.
- Patch : cache Redis + invalidation écrite.
- Mesure : hit ratio projeté > 80 % (N3.10 — anti-règle : jamais avant de
  connaître le hit ratio).

**12. Timeouts sur les API externes**
- Vérif : grep des appels sortants sans timeout.
- Patch : timeout explicite + repli + disjoncteur sur les dépendances critiques.
- Mesure : **toujours justifié** — un appel sortant sans timeout est un défaut
  en soi, pas besoin d'autre preuve.

---

## Infrastructure

**6. Fichiers statiques via CDN**
- Vérif : statiques servis par l'origine ou un CDN ? hit ratio CDN.
- Patch : CDN devant les statiques.
- Mesure : TTFB correct mais statiques lents/loin (anti-règle : un CDN devant un
  serveur lent ne corrige pas le TTFB).

**18. Serveurs près de la base**
- Vérif : latence app↔DB.
- Patch : colocaliser app et base (même région/zone).
- Mesure : latence app↔DB > 5 ms (N4.2).

**27. Load balancer (répartir le trafic)**
- Vérif : combien d'instances ? bascule testée ?
- Patch : répartiteur sur ≥ 2 instances + test de bascule.
- Mesure : instance unique, ou saturation en pointe (N4.5/N4.6).

**11. Virer les dépendances inutiles**
- Vérif : analyse du bundle (front) / inventaire des paquets (back) ; poids et
  usage réel.
- Patch : suppression des dépendances mortes ou surdimensionnées.
- Mesure : dépendance pesant dans le bundle et non utilisée, ou doublon.

---

## Mesure (les deux dernières — ce sont elles qui rendent les 28 autres
possibles)

**29. Mesurer la vitesse réelle des utilisateurs (RUM)**
- Vérif : un RUM collecte-t-il LCP/INP/CLS/TTFB sur le trafic réel, segmenté ?
- Patch : poser le RUM (N0.1/N0.2) **avant toute autre optimisation**.
- Mesure : **préalable obligatoire** — c'est l'étape 1 du skill, pas une option.

**30. Tester sur téléphones et devices lents**
- Vérif : les mesures viennent-elles de devices réels bas de gamme, pas d'un
  laptop ?
- Patch : bridage CPU 4×/réseau 4G lente en lab, et un device réel milieu de
  gamme à chaque version majeure (N5.6).
- Mesure : **préalable de méthode** — sans ça tout chiffre de N1 est faussé par
  un matériel trop rapide.

---

## Tableau de prise en charge (à remplir par projet)

```
Parcours audité : ______   Verdict N1 : LCP __ INP __ CLS __ TTFB __ API p95 __

#   Directive                     État            Gain mesuré   Raison si non
1   Squelettes                    OK/PATCH/NJ     —/+__ms       
2   WebP/AVIF                      ...
3   Dédoublonnage API
4   Debounce recherche
5   Pagination API
6   CDN statiques
7   File de jobs
8   N+1
9   Compression
10  Redis données chaudes
11  Dépendances inutiles
12  Timeouts API externes
13  Streaming IA
14  UI optimiste
15  Lazyload images
16  Requêtes parallèles
17  Index DB
18  Colocalisation DB
19  SSR/SSG
20  Scripts tiers différés
21  Découpage JS
22  Virtualisation listes
23  Colonnes utiles
24  Pool de connexions
25  Re-renders
26  Cache navigateur
27  Load balancer
28  Polices
29  RUM                           (préalable)
30  Devices lents                 (préalable)

Règle : une ligne PATCH n'existe que si une mesure la désigne. Une ligne NJ
(non justifiée) doit porter sa raison. Aucune ligne ne reste vide.
```
