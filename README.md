# Skills — contrôle de projet

Skills de contrôle qualité d'un projet web avant mise en production.

| Skill | Rôle |
|-------|------|
| `controle-projet/` | Orchestrateur : enchaîne perf puis sécurité, rend un go/no-go unique. |
| `perf-projet/` | Les 30 optimisations de performance : vérifier et patcher sur mesure. |
| `secu-projet/` | 9 domaines de sécurité + pipeline CI + anti-bots + pentest de pré-prod. |
| `regles-perf.md` | Référentiel de seuils (LCP, INP, CLS, TTFB, p75…) appliqué par `perf-projet`. |

Principe commun : **pas de verdict sans preuve**. Côté perf, on ne patche que
ce qu'une mesure désigne ; côté sécu, on raisonne en couverture de surface, pas
en score.
