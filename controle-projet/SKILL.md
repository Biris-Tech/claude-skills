---
name: controle-projet
description: >
  Contrôle complet d'un projet web avant mise en production : orchestre la perf
  puis la sécurité en déléguant aux deux skills dédiés. À déclencher quand
  l'utilisateur veut un « contrôle complet », « audit complet du projet »,
  « vérifier perf ET sécu », « préparer la mise en prod de A à Z », « revue
  avant lancement », ou enchaîner les deux. Pour un besoin purement perf, utiliser
  perf-projet ; purement sécu, secu-projet. Ce skill ne duplique aucun contenu :
  il séquence et rend un go/no-go unique.
---

# controle-projet — orchestrateur perf + sécurité

Point d'entrée unique pour un contrôle de projet de bout en bout. Il **ne
contient pas** les tests : il séquence les deux skills dédiés et consolide le
verdict. C'est le choix « orchestrateur léger » — l'entrée unique sans fusionner
les contenus ni brouiller le déclenchement de chaque domaine.

## Ce qu'il délègue

| Phase | Skill | Contenu |
|-------|-------|---------|
| Perf | `perf-projet` ([../perf-projet/SKILL.md](../perf-projet/SKILL.md)) | les 30 directives : mesurer → verdict → profiler → patcher sur mesure → verrouiller |
| Sécu | `secu-projet` ([../secu-projet/SKILL.md](../secu-projet/SKILL.md)) | 9 domaines + pipeline CI + bots + pentest pré-prod, derrière garde d'autorisation |

Le référentiel de seuils perf est [../regles-perf.md](../regles-perf.md).

## Ordre d'exécution

**Perf d'abord, sécu ensuite.** Raison : certains correctifs perf changent la
surface à sécuriser — on ajoute un CDN, un cache Redis, une file de jobs, un
endpoint de pagination, un load balancer. Les auditer sécu *après* les avoir
posés évite de repasser la sécu deux fois.

```
0. Cartographier le projet (une seule fois, partagée par les deux phases)
1. PERF   → invoquer perf-projet, dérouler ses étapes 1→5
2. SÉCU   → invoquer secu-projet, dérouler ses étapes 0→4
             (la garde d'autorisation s'arme ICI, pas avant)
3. Go/no-go consolidé
```

Chaque phase garde **ses propres règles** intactes :
- Perf : *pas de patch sans mesure* (on ne pose Redis/index/file de jobs que si
  une mesure les désigne).
- Sécu : *couverture de surface, pas de score* ; la garde d'autorisation (cible
  à l'utilisateur, instance de test, scans actifs cadrés) ne s'applique qu'à la
  phase sécu.

## Déroulé

### Phase 0 — Cartographie partagée
Pile, framework, base, hébergeur, CI, CDN/WAF, surface exposée, parcours
critique. Faite une fois, réutilisée par les deux skills (ne pas la refaire dans
chaque phase).

### Phase 1 — Perf (déléguée à perf-projet)
Suivre perf-projet de l'instrumentation au verrouillage CI. Livrable : verdict
N1 segmenté, liste des coûts en ms, tableau des 30 (OK/PATCH/NJ), correctifs
avec gain avant/après, budget perf en CI.

### Phase 2 — Sécu (déléguée à secu-projet)
**Passer la garde d'autorisation d'abord** (cf. secu-projet). Puis dérouler les
9 domaines, le pipeline sécu, le volet bots, le pentest de pré-prod. Livrable :
fiche de couverture, failles triées par exploitabilité × donnée exposée,
correctifs avec preuve avant/après.

### Phase 3 — Go/no-go consolidé
Un seul verdict pour la mise en prod, qui agrège les deux :

- **No-go** si l'un des deux bloque : une faille sécu critique/élevée non
  corrigée, **ou** une métrique N1 en zone « lent » sur le parcours critique.
- **Go sous conditions** : correctifs moyens planifiés avec dates, et détection
  en place pour les couvrir.
- **Go** : sécu sans critique/élevé + pipelines verts + perf dans les seuils +
  détection et sauvegarde-restaurée opérationnelles.

Conclure par une phrase nette, pas un décompte d'alertes ni de millisecondes.

## Livrable unique
Un **rapport de contrôle** en deux sections (perf, sécu) — reprenant les
livrables de chaque skill — et une **décision go/no-go** en tête. Les journaux
de correctifs des deux phases y sont annexés.

## Quand NE PAS utiliser ce skill
- Besoin uniquement perf → `perf-projet` directement (plus léger, pas de garde
  sécu inutile).
- Besoin uniquement sécu → `secu-projet` directement.
Ce skill n'a d'intérêt que pour l'enchaînement complet avant un lancement.
